# hyranse_workspace

Конфигурация multi-root workspace для проектов Hyranse.

## Использование

Откройте `hyranse_workspace.code-workspace` в Cursor/VS Code, чтобы работать со всеми репозиториями Hyranse в одном окне.

## GitHub MCP

Конфигурация GitHub MCP лежит в [`.cursor/mcp.json`](.cursor/mcp.json) и подхватывается Cursor при открытии workspace.

**Требования:** Docker Desktop (запущен), Cursor v0.48+.

Токен задаётся через переменную окружения `GITHUB_TOKEN` (см. `~/.zshrc`).

После открытия workspace перезапустите Cursor и проверьте: **Settings → Tools & Integrations → MCP Tools** (у `github` должен быть зелёный статус).
