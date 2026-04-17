# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Проект

AR-виджет примерки цвета губ для Makeup Kitchen. Встраивается в Tilda через `<iframe>`. Один файл, никакой сборки.

**Хостинг:** GitHub Pages → https://podrozdel.github.io/lip-tryon/

## Деплой

```bash
git add index.html watermark.svg
git commit -m "описание изменений"
git push
# GitHub Pages обновляется ~1 минуту
```

После пуша обязательно поднять версию кэша в двух местах:
1. `index.html` — комментарий `<!-- cache bust: vN -->`
2. Tilda HTML-блок — `?v=N` в src iframe

## Файлы

- `index.html` — всё приложение (CSS + HTML + JS в одном файле)
- `watermark.svg` — логотип на селфи (126×126px). Менять можно без правок кода — просто заменить файл и запушить.

## Архитектура

Один ES-модуль (`<script type="module">`). Функции доступные из HTML-атрибутов (`onclick`) обязательно должны быть назначены на `window`:

```js
window.onAgeCheck = function() { ... }
window.startWithConsent = function() { ... }
window.takeSelfie = async function() { ... }
```

**Поток работы:**
1. Страница загружается → показывается `#age-gate` (чекбокс 18+)
2. Пользователь ставит галочку → активируется кнопка
3. Клик → `startWithConsent()` → скрывается age-gate, показывается `#section-camera`, вызывается `startCamera()`
4. `startCamera()` → `initMediaPipe()` → `requestAnimationFrame(cameraLoop)`
5. `cameraLoop` каждые 3 кадра запускает `faceLandmarker.detectForVideo()` → координаты губ → `drawLipColor()`

**MediaPipe:** `FaceLandmarker` из `@mediapipe/tasks-vision@0.10.14` через CDN jsDelivr. Модель (~30MB) грузится с `storage.googleapis.com` один раз. Все данные обрабатываются локально в браузере — на сервер ничего не уходит.

**Наложение цвета:** offscreen canvas + `clip(lipPath, 'evenodd')` + blend mode `color` + `blur(2px)` для мягких краёв. Блеск — radial gradient + blend mode `screen`.

**Индексы лендмарков губ (478-точечная сетка):**
```js
const OUTER = [61,185,40,39,37,0,267,269,270,409,291,375,321,405,314,17,84,181,91,146];
const INNER = [78,191,80,81,82,13,312,311,310,415,308,324,318,402,317,14,87,178,88,95];
```

**Селфи:** на мобиле — `navigator.share({files})` (iOS share sheet), на десктопе — скачивание через `<a download>`. Watermark рисуется из `watermark.svg` через `Image` → `drawImage` на canvas.

## Дизайн-система

```css
--brand: #3c0e2c   /* тёмно-бордовый */
--accent: #cf66ab  /* розовый */
--bg: #fff5f5      /* фон страницы */
```

Шрифты: **Montserrat** (заголовки, кнопки) + **Manrope** (UI текст) — Google Fonts.

Десктоп (≥768px): двухколоночный grid (камера слева, контролы справа). Media query — в **конце** CSS, после всех глобальных стилей (иначе глобальные стили перебивают).

## Вставка в Tilda

```html
<iframe 
  src="https://podrozdel.github.io/lip-tryon/?v=17" 
  width="100%" 
  frameborder="0"
  allow="camera; web-share"
  style="display:block; border:none; height:100vh; width:100%;"
></iframe>
```

`allow="web-share"` обязателен для работы iOS share sheet внутри iframe.
