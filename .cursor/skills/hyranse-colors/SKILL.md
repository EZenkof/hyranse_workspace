---
name: hyranse-colors
description: >-
  Hyranse colors, CSS variables, light/dark themes. Use when styling UI,
  emails, plugins, or brand colors.
disable-model-invocation: true
---

# Hyranse — цветовая система

Источник истины: `hyranse_frontend_main/src/constants.scss` (light) и `hyranse_frontend_main/src/index.scss` (dark overrides).

## Принципы

1. **Всегда CSS-переменные** — не хардкодить hex в компонентах. Исключения: `#ffffff` на primary-кнопках, редкие SVG fill.
2. **Палитра Tailwind-совместимая**: Slate (нейтрали), Sky (бренд), Emerald (успех/OTW), Rose (ошибки), Amber (Premium/PRO).
3. **Два режима**: `data-theme="light"` (default) и `data-theme="dark"` на `<html>`. Переключение — `useTheme` hook.
4. **B2B-стиль**: мягкие slate-оттенки вместо чистого чёрного, sky blue как primary, лёгкие тени и 8–12px border-radius.

## Бренд (Primary — Sky Blue)

| Роль | Light | Dark | Переменная |
|------|-------|------|------------|
| Primary | `#0284c7` Sky-600 | `#38bdf8` Sky-400 | `--main-color` |
| Hover | `#0369a1` Sky-700 | `#0ea5e9` Sky-500 | `--main-color-hover` |
| Tint/bg | `rgba(2,132,199,0.06)` | `rgba(56,189,248,0.12)` | `--main-second-color` |
| Border | `rgba(2,132,199,0.15)` | `rgba(56,189,248,0.2)` | `--main-color-border` |
| Focus ring | `rgba(2,132,199,0.1)` | `rgba(56,189,248,0.15)` | `--main-color-focus-ring` |
| Shadow | `rgba(2,132,199,0.15)` | `rgba(56,189,248,0.2)` | `--main-color-shadow` |

**Когда использовать**: CTA-кнопки, активные пункты меню, ссылки, focus states, фильтры (include), selection highlight.

## Семантические цвета

| Смысл | Light | Dark | Переменная |
|-------|-------|------|------------|
| Успех / OpenToWork / Open | `#059669` Emerald-600 | `#34d399` Emerald-400 | `--green` |
| Ошибка / Exclude / Validation | `#e11d48` Rose-600 | `#fb7185` Rose-400 | `--red` |
| Premium / PRO / Paused | `#d97706` Amber-600 | `#fbbf24` Amber-400 | `--amber` |
| Premium tint | `rgba(217,119,6,0.08)` | `rgba(251,191,36,0.12)` | `--amber-second-color` |
| Text highlight | `#dbeafe` Blue-100 | `#0c4a6e` Sky-900 | `--highlight` |

**Градиенты** (для badge/карточек): `--green-gradient`, `--error-gradient`, `--blue-gradient` (+ hover-варианты).

## Нейтрали (фон, текст, границы)

### Light
- Фон основной: `--main-background-color` `#ffffff`
- Фон карточек/панелей: `--second-background-color` `#f0f4f8`
- Фон инпутов: `--third-background-color` `#f8fafc`
- Заголовки: `--black` `#0f172a` (Slate-900)
- Текст: `--font-color` `#1e293b` (Slate-800)
- Вторичный текст: `--second-font-color` `#64748b` (Slate-500)
- Граница: `--border-color` `#e2e8f0` (Slate-200)
- Backdrop: `--backdrop-сolor` `#f1f5f9` (Slate-100)

### Dark
- Фон основной: `--main-background-color` `#0b0f19`
- Карточки: `--second-background-color` `#171e30`
- Инпуты: `--third-background-color` `#111827`
- Заголовки: `--black` `#f8fafc` (Slate-50)
- Текст: `--font-color` `#f1f5f9` (Slate-100)
- Вторичный: `--second-font-color` `#94a3b8` (Slate-400)
- Граница: `--border-color` `#1e293b` (Slate-800)

## Типографика (связанные токены)

- Бренд/заголовки: `--font-family-brand` → `'Montserrat', sans-serif`
- UI/текст: `--font-family-ui` → `'Inter', -apple-system, ...`
- Размеры: `--font-xxl`, `--font-xl`, `--font-size`, `--font-s`, `--font-xs` (responsive breakpoint 1070px)

## Паттерны использования в UI

### Primary button
```scss
background-color: var(--main-color);
color: #ffffff;
&:hover { background-color: var(--main-color-hover); }
```

### Input
```scss
background: var(--third-background-color);
border: 1px solid var(--border-color);
&:focus {
  border-color: var(--main-color);
  box-shadow: 0 0 0 4px var(--main-color-focus-ring);
}
// Ошибка:
border-color: var(--red);
background: rgba(244, 63, 94, 0.02);
```

### Filter chips
- Include: `--main-color` + `--main-second-color` + `--main-color-border`
- Exclude: `--red` + `rgba(244,63,94,0.08)` + `rgba(244,63,94,0.15)`

### Status badges (вакансии)
- Open: `--green` + `--green-gradient`
- Paused: `--amber` + `--amber-second-color`
- Closed: `--second-font-color` + `--disable-color`

### Карточки
```scss
background: var(--main-background-color);
border: 1px solid var(--border-color);
box-shadow: var(--box-shadow);
&:hover { border-color: var(--main-color); }
```

### Selection
```scss
::selection {
  background-color: var(--main-second-color);
  color: var(--main-color);
}
```

## Адаптация в других проектах workspace

При работе вне React-фронта (WordPress landing, email plugin, wiki) — **сохранять те же hex-значения**:

| Frontend token | Landing alias (WP) |
|----------------|-------------------|
| `--main-color` | `--main-color` / `#0284c7` |
| `--main-color-hover` | `--main-hover` / `#0369a1` |
| `--main-second-color` | `--main-tint` / `rgba(2,132,199,0.08)` |
| `--main-background-color` | `--bg` / `#ffffff` |
| `--second-background-color` | `--bg-soft` / `#f0f4f8` |
| `--black` | `--text` / `#0f172a` |
| `--font-color` | `--text-body` / `#1e293b` |
| `--second-font-color` | `--text-muted` / `#64748b` |
| `--green` | `--green` / `#059669` |
| `--amber` | `--amber` / `#d97706` |

LinkedIn-интеграция (landing): `--linkedin: #0077b5` — единственный внешний бренд-цвет.

## Чего избегать

- Чистый `#000000` для текста — использовать `--black` / Slate-900
- Случайные синие (`#1e90ff`, `#0066cc`) — только Sky-палитра бренда
- Новые цвета без семантической переменной
- Игнорирование dark theme при добавлении новых токенов

## Чеклист при добавлении стилей

- [ ] Используются существующие CSS-переменные
- [ ] Семантика совпадает (green=успех, red=ошибка, amber=premium)
- [ ] Проверены оба режима light/dark
- [ ] Шрифты: Montserrat для заголовков, Inter для UI
- [ ] Border-radius: 8px (инпуты), 12px (контейнеры)

## Дополнительно

Полная таблица всех токенов: [reference.md](reference.md)
