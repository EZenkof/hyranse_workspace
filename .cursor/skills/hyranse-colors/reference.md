# Hyranse — полный справочник CSS-переменных

Файлы-источники:
- `hyranse_frontend_main/src/constants.scss` — `:root` (light)
- `hyranse_frontend_main/src/index.scss` — `[data-theme='dark']`

## Primary (Sky Blue)

| Variable | Light | Dark |
|----------|-------|------|
| `--main-color` | `#0284c7` | `#38bdf8` |
| `--main-color-hover` | `#0369a1` | `#0ea5e9` |
| `--main-color-shadow` | `rgba(2,132,199,0.15)` | `rgba(56,189,248,0.2)` |
| `--main-color-focus-ring` | `rgba(2,132,199,0.1)` | `rgba(56,189,248,0.15)` |
| `--main-color-border` | `rgba(2,132,199,0.15)` | `rgba(56,189,248,0.2)` |
| `--main-color-hover-bg` | `rgba(2,132,199,0.12)` | `rgba(56,189,248,0.15)` |
| `--main-color-border-strong` | `rgba(2,132,199,0.3)` | `rgba(56,189,248,0.35)` |
| `--main-second-color` | `rgba(2,132,199,0.06)` | `rgba(56,189,248,0.12)` |
| `--scroll-color` | `#0284c7` | *(не переопределён, наследует light)* |

## Semantic

| Variable | Light | Dark | Назначение |
|----------|-------|------|------------|
| `--green` | `#059669` | `#34d399` | Успех, OpenToWork, статус Open |
| `--red` | `#e11d48` | `#fb7185` | Ошибки, exclude-фильтры |
| `--amber` | `#d97706` | `#fbbf24` | Premium/PRO, статус Paused |
| `--amber-second-color` | `rgba(217,119,6,0.08)` | `rgba(251,191,36,0.12)` | Фон amber-badge |
| `--highlight` | `#dbeafe` | `#0c4a6e` | Подсветка текста/опыта |

## Backgrounds

| Variable | Light | Dark |
|----------|-------|------|
| `--main-background-color` | `#ffffff` | `#0b0f19` |
| `--second-background-color` | `#f0f4f8` | `#171e30` |
| `--third-background-color` | `#f8fafc` | `#111827` |
| `--backdrop-сolor` | `#f1f5f9` | `#060913` |
| `--focus-color` | `#f8fafc` | `#111827` |
| `--disable-color` | `rgba(15,23,42,0.04)` | `rgba(30,41,59,0.5)` |

## Text

| Variable | Light | Dark |
|----------|-------|------|
| `--black` | `#0f172a` | `#f8fafc` |
| `--font-color` | `#1e293b` | `#f1f5f9` |
| `--second-font-color` | `#64748b` | `#94a3b8` |
| `--transparent-color` | `rgba(100,116,139,0.3)` | `rgba(148,163,184,0.2)` |

## Borders

| Variable | Light | Dark |
|----------|-------|------|
| `--border-color` | `#e2e8f0` | `#1e293b` |
| `--border-focus-color` | `#cbd5e1` | `#334155` |
| `--second-border-color` | `#f1f5f9` | `#111827` |

## Gradients

| Variable | Значение (light) |
|----------|------------------|
| `--error-gradient` | white → `rgba(225,29,72,0.05)` |
| `--error-gradient-hover` | white → `rgba(225,29,72,0.1)` |
| `--green-gradient` | white → `rgba(5,150,105,0.05)` |
| `--green-gradient-hover` | white → `rgba(5,150,105,0.1)` |
| `--blue-gradient` | white → `rgba(2,132,199,0.08)` |
| `--skeleton` | `#f1f5f9` ↔ `#f8fafc` shimmer (dark: `#1e293b` ↔ `#334155`) |

## Effects & Layout (theme-independent)

| Variable | Value |
|----------|-------|
| `--box-shadow` | light: subtle slate shadow; dark: `rgba(0,0,0,0.4)` |
| `--transition` | `0.2s cubic-bezier(0.4, 0, 0.2, 1)` |
| `--input-border-radius` | `8px` |
| `--border-radius` | `12px` |
| `--header-height` | `64px` |
| `--svg-width` | `20px` |
| `--gap` | `8px` |
| `--page-gap` | `30px` |

## Typography

| Variable | Desktop (≥1070px) | Mobile (<1070px) |
|----------|-------------------|------------------|
| `--font-xxl` | 24px | 26px |
| `--font-xl` | 20px | 22px |
| `--font-size` | 14px | 16px |
| `--font-s` | 13px | 14px |
| `--font-xs` | 11px | 12px |

| Variable | Value |
|----------|-------|
| `--font-family-brand` | `'Montserrat', sans-serif` |
| `--font-family-ui` | `'Inter', -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif` |

## Tailwind-эквиваленты (для справки)

```
Sky-600  #0284c7  → primary light
Sky-700  #0369a1  → primary hover light
Sky-400  #38bdf8  → primary dark
Slate-900 #0f172a → headings light
Slate-800 #1e293b → body text light
Slate-500 #64748b → muted text light
Slate-200 #e2e8f0 → borders light
Emerald-600 #059669 → success light
Rose-600  #e11d48 → error light
Amber-600 #d97706 → premium light
```

## Theme API

```tsx
// hyranse_frontend_main/src/common/helper/hooks/useTheme.tsx
document.documentElement.setAttribute('data-theme', 'light' | 'dark');
localStorage.setItem('theme', theme);
// Default: system prefers-color-scheme, fallback light
```
