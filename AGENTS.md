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

> ⏳ **Пока недоступно.** MCP-сервер написан в движке (6 инструментов:
> `scan_repo`, `scan_diff`, `list_findings`, `get_finding`, `verify_finding`,
> `list_detectors`), но в публичный образ не входит — `fastmcp` нет в
> scanner-зависимостях и команды `gsc mcp` не существует. Конфигурации для
> подключения нет, пока сервер не доедет до образа. Не советуй пользователям
> настраивать MCP.

Когда доедет: транспорт `stdio` (локальный, без auth) или `http` (требует
`GSC_MCP_TOKEN`). Деструктивных действий (auto-patch/PR) в MCP не будет —
осознанно, они остаются в CLI/human-лупе.

> ⚠️ `verify_finding` перезапускает PoC в sandbox — по побочным эффектам это не
> «чисто read-only» (исполняет код). Sandbox: rlimit-лимиты (CPU/RAM/procs/fsize),
> whitelist env, опционально контейнер с egress-deny и read-only rootfs.

## Лицензия

Этот репозиторий (Action + документация) = **Apache 2.0**. Движок
(docker-образ `ghcr.io/poliakarmai/gsc-scanner`) = **BSL 1.1** — сканировать
свои репозитории можно всегда; коммерческий SaaS/SAST-сервис в конкуренцию —
только по коммерческой лицензии. Change Date 2030-08-06 → движок переходит на Apache 2.0.
