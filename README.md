# hyranse_workspace

Конфигурация multi-root workspace для проектов Hyranse.

## Использование

Откройте `hyranse_workspace.code-workspace` в Cursor/VS Code, чтобы работать со всеми репозиториями Hyranse в одном окне.

## GitHub MCP

Конфигурация GitHub MCP лежит в [`.cursor/mcp.json`](.cursor/mcp.json) и подхватывается Cursor при открытии workspace.

**Требования:** Docker Desktop (запущен), Cursor v0.48+.

Токен задаётся через переменную окружения `GITHUB_TOKEN` (см. `~/.zshrc`).

После открытия workspace перезапустите Cursor и проверьте: **Settings → Tools & Integrations → MCP Tools** (у `github` должен быть зелёный статус).

## Playwright MCP

Конфигурация Playwright MCP лежит в [`.cursor/mcp.json`](.cursor/mcp.json) рядом с GitHub MCP.

**Требования:** Node.js 18+ и `npx` в PATH.

Сервер запускается через `npx -y @playwright/mcp@latest` в headless Chromium. Перед первым использованием установите браузер:

```bash
npx @playwright/mcp@latest install-browser chrome-for-testing
```

После изменения конфига перезагрузите окно Cursor (**Cmd+Shift+P → Reload Window**) и проверьте статус `playwright` в **Settings → Tools & Integrations → MCP Tools**.
