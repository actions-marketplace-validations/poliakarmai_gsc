# AGENTS.md — GSC для AI-агентов

GSC — SAST-сканер: 69 детекторов (SAST · SCA · Secrets · IaC), PoC (доказанный
эксплойт), Proof-of-Fix (верифицированный фикс), self-healing CI.

## Как подключить GSC

### Вариант A — GitHub Action (для репозиториев на GitHub)

Добавь `.github/workflows/gsc.yml`:

```yaml
name: GSC Audit
on:
  pull_request:
    branches: [main, master]
jobs:
  audit:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: poliakarmai/gsc@v1
        with:
          fail_on_critical: false
          # fail_on_score: 80      # опционально: упасть, если score ниже порога
```

Скан запускается автоматически на каждый PR: комментарий с находками
(CRITICAL/HIGH) + JSON-отчёт. Опционально — BYO-LLM ключ для ревалидации
(`llm_api_key`).

### Вариант B — MCP (для Claude Code / Cursor / Codex / Hermes)

Движок отдаёт 6 инструментов по MCP (read-only по отношению к репозиторию; `verify_finding` исполняет PoC): `scan_repo` (скан репо),
`scan_diff` (только изменённые файлы), `list_findings` (из БД, без скана),
`get_finding` (по ключу), `verify_finding` (перезапуск PoC в sandbox),
`list_detectors` (список детекторов).

Сервер внутри образа, отдельная установка не нужна:

```json
{
  "mcpServers": {
    "gsc": {
      "command": "docker",
      "args": ["run", "--rm", "-i",
               "-v", "${PWD}:/repo:ro",
               "-v", "gsc-state:/app/.hermes/state",
               "ghcr.io/poliakarmai/gsc-scanner:v1", "mcp"],
      "env": { "GSC_ALLOWED_ROOTS": "/repo" }
    }
  }
}
```

Это `.mcp.json` (Claude Code) или `~/.claude.json` (глобально). Cursor — та же
форма. HTTP-транспорт требует `GSC_MCP_TOKEN` (fail-closed).

Деструктивных действий (auto-patch/PR) в MCP нет — осознанно, они остаются
в CLI/human-лупе.

> ⚠️ Честно о границах (проверено в пересобранном образе):
> - `verify_finding` **исполняет PoC**. Гейты в коде реальные (DESTRUCTIVE и
>   недетерминированные PoC отклоняются), но **OS-изоляции нет**: в образе нет
>   docker/podman, поэтому PoC идёт в том же контейнере под rlimit, а статус
>   помечается `unverified`, а не `verified`. Изоляция — только с проброшенным
>   `docker.sock` и образом `gsc-sandbox`.
> - `scan_repo` — единственный тул, который реально сканирует, и он же сохраняет
>   находки в БД (`persisted: true`). `list_findings` и `get_finding` видят их
>   сразу; `get_finding` отвечает из БД (`source: "database"`). Пересканирование
>   (~30–60 с) — только когда ключа в БД нет.
> - Нужен writable volume на `/app/.hermes/state` (там SQLite). `--read-only` не
>   использовать — БД молча не создастся.
> - `GSC_ALLOWED_ROOTS` ограничивает `scan_repo`/`scan_diff` смонтированными путями.

## Лицензия

Этот репозиторий (Action + документация) = **Apache 2.0**. Движок
(docker-образ `ghcr.io/poliakarmai/gsc-scanner`) = **BSL 1.1** — сканировать
свои репозитории можно всегда; коммерческий SaaS/SAST-сервис в конкуренцию —
только по коммерческой лицензии. Change Date 2030-08-06 → движок переходит на Apache 2.0.
