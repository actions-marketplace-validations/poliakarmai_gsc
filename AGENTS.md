# AGENTS.md — GSC для AI-агентов

GSC — SAST-сканер: 70 детекторов (SAST · SCA · Secrets · IaC), PoC (доказанный
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

Движок отдаёт 6 инструментов по MCP (read-only по отношению к репозиторию; `verify_finding` исполняет PoC в bubblewrap-песочнице внутри образа, см. границы ниже): `scan_repo` (скан репо),
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
               "--security-opt", "seccomp=unconfined",
               "--security-opt", "apparmor=unconfined",
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

> ⚠️ Честно о границах (проверено в пересобранном образе 25.10.2026):
> - `verify_finding` исполняет PoC **внутри образа** через bubblewrap: mount/pid/net/user
>   namespaces, пустой env, read-only системные деревья, writable только workspace,
>   `--die-with-parent`. Docker/podman-демон не нужен. Живой прогон: `NET_DENIED`,
>   uid 999, `/repo` из песочницы не виден.
> - **Нужны два флага контейнера:** `--security-opt seccomp=unconfined --security-opt apparmor=unconfined`.
>   Без них ядро не даёт создать user-namespace и тул отвечает `status: "unavailable"` —
>   честно (PoC не запускался), а **не** «не воспроизводится».
>   Гейты в коде реальные: DESTRUCTIVE и недетерминированные PoC отклоняются.
> - Проброс `docker.sock` в этот контейнер — не изоляция, а выдача PoC хостового
>   демона (эквивалент root): не делать.
> - `scan_repo` ограничен: свип детекторов — `GSC_SCAN_TIMEOUT` (default 300 с),
>   инвентарь файлов — `GSC_MAX_FILES` (default 50 000). При упоре в лимит ответ
>   приходит со `status: "partial"`, `truncated: true` и списком детекторов,
>   которые не запускались — мало находок там **не значит** «чисто».
>   Truncated-скан в БД не сохраняется.
> - `scan_repo` — единственный тул, который реально сканирует, и он же сохраняет
>   находки в БД (`persisted: true`). `list_findings` и `get_finding` видят их
>   сразу; `get_finding` отвечает из БД (`source: "database"`). Пересканирование
>   (~30–60 с) — только когда ключа в БД нет.
> - Нужен **writable** volume на `/app/.hermes/state` (там SQLite). Docker создаёт
>   его от root, а процесс идёт под uid 999 — сначала
>   `docker run --rm -v gsc-state:/s --user 0 <image> chown -R 999:999 /s`.
>   Иначе `persisted: false` (скан есть, БД нет). `--read-only` не подходит.
> - `GSC_ALLOWED_ROOTS` ограничивает `scan_repo`/`scan_diff` смонтированными путями.

## Лицензия

Этот репозиторий (Action + документация) = **Apache 2.0**. Движок
(docker-образ `ghcr.io/poliakarmai/gsc-scanner`) = **BSL 1.1** — сканировать
свои репозитории можно всегда; коммерческий SaaS/SAST-сервис в конкуренцию —
только по коммерческой лицензии. Change Date 2030-08-06 → движок переходит на Apache 2.0.
