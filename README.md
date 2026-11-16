# canvas-window-resize

Аддон к [canvas-tty-plugins](https://github.com/justNothing228/canvas-tty-plugins)
(и к любой новой версии resize-плагина): изменение размера окон на `Ctrl` + `+` / `-`.

Один файл (`canvas-window-resize.js`), без зависимостей и сборки. Он сам определяет,
где запущен, и работает в трёх контекстах:

| Контекст | Как определяется | Что делает |
|---|---|---|
| Окно CanvasTTY | есть `window.canvasTTY` (или `file:`-документ) | выделяет карточки плагинов кликом и меняет размер выделенного слоя |
| Карточка плагина на канвасе | `window.parent !== window` | пересылает горячие клавиши хосту, тот меняет размер карточки |
| Отдельное окно Electron | ничего из вышеперечисленного | меняет размер самого окна (`window.resizeTo`/`moveTo`) |

Горячие клавиши:

- `Ctrl` + `+` / `-` — пропорционально;
- `Ctrl` + стрелка, затем `+` / `-` — наращивание в сторону стрелки
  (противоположный край неподвижен);
- шаг `1.15`, границы `420×260 … 1600×1100`, стрелка действует 1.2 с.

## Встраивание

### 1. В набор canvas-tty-plugins

```jsonc
// plugins.json
{
  "hook": "hooks/canvastty-plugins.js",
  "addons": ["addons/canvas-window-resize.js"]   // ← этот файл, подключённый до хука
}
```

`loader.py` подключаяет аддон **до** хука, поэтому хук должен уступать ему
`Ctrl` + `+` / `-`:

```js
// hooks/canvastty-plugins.js — в обработчике клавиш
if (window.__canvasWindowResizeActive) return;   // аддон владеет Ctrl + '+' / '-'
```

Фолбэк «ничего не выделено → зум канваса» передаётся через `configure`:

```js
window.CanvasWindowResize.configure({
  onEmptySelection: plus => zoomCanvas(plus ? 1 : -1)
});
```

### 2. В свой плагин (вендоренная копия)

```html
<script src="canvastty-plugin://host/sdk.js"></script>
<script src="canvas-window-resize.js"></script>   <!-- копия этого файла -->
<script src="app.js"></script>
```

Аддон установится сам и выберет контекст: в карточке на канвасе он пересылает
клавиши хосту, в отдельном окне Electron — меняет размер окна.

### 3. В новую версию resize-плагина

Скопируйте файл в репозиторий плагина, подключите перед кодом resize-функции и
оставьте в старом коде проверку `window.__canvasWindowResizeActive` (см. пункт 1).
API аддона при этом доступен как `window.CanvasWindowResize`.

## API

```js
window.CanvasWindowResize = {
  install(options),                 // обычно не нужен: установка автоматическая
  configure(options),               // изменить опции после установки
  plan(bounds, factor, dir),        // чистая математика: {position, size}
  selectedLayers(),                 // выделенные слои (только в контексте host)
  resizeSelected(factor, dir),      // изменить размер выделенного
  resizeElectronWindow(factor, dir),
  context()                         // "host" | "frame" | "window"
};
```

Опции (`install(options)` / `configure(options)`):

| Опция | По умолчанию | Смысл |
|---|---|---|
| `factor` | `1.15` | шаг изменения размера |
| `minWidth` / `minHeight` | `420` / `260` | минимальный размер |
| `maxWidth` / `maxHeight` | `1600` / `1100` | максимум (для карточек его жёстко задаёт приложение) |
| `arrowWindowMs` | `1200` | сколько миллисекунд стрелка «вооружена» |
| `messageSource` | `"canvas-window-resize"` | источник `postMessage` от карточки |
| `extraSources` | `["canvastty-plugin-hotkey"]` | дополнительные принимаемые источники |
| `onEmptySelection` | `null` | фолбэк, когда цель не найдена (например, зум канваса) |
| `onGrowClamped` | `null` | все цели уже на потолке |
| `notify` | `true` | показывать новый размер внизу экрана |

## Протокол сообщений

Карточка на канвасе шлёт хосту (и наоборот) объекты в `window.postMessage`:

```js
{ source: "canvas-window-resize", type: "activate" }                        // клик в карточке
{ source: "canvas-window-resize", type: "resize", factor: 1.15, direction: "right" }
```

`direction`: `"left" | "right" | "up" | "down" | null`.

## Важные детали (уже учтены)

- **Карточку нельзя ресайзить через `settings.update`.** CanvasTTY хранит размер в
  своём React-состоянии и перерисовывает карточку только при вызове её
  `onBoundsChange`. Поэтому аддон достаёт обработчик из React-fiber и вызывает его —
  как будто пользователь потянул за ручку ресайза.
- **Отдельное окно Electron врёт о своём размере**: `outerWidth`/`outerHeight`
  отдают размер, с которым окно создавалось. Поэтому аддон помнит «логический»
  размер в `localStorage['canvas-window-resize.size']`.
- **Синглтон**: загрузчик может оценить файл дважды — аддон ставит
  `window.__canvasWindowResizeActive` и `window.__canvasWindowResizeSingleton`,
  чтобы фактор не применился дважды.
- **Потолок 1600×1100** для карточек плагинов жёстко задан самим приложением:
  установка 2200×1500 откатывается до 1600×1100.

## Лицензия

MIT — см. [LICENSE](LICENSE).
