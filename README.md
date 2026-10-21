# 🛡️ GSC — Git Security Checker

[![GSC on GitHub Marketplace](https://img.shields.io/badge/Marketplace-GSC%20Security%20Audit-blue)](https://github.com/marketplace/actions/gsc-security-audit)
[![HOL Guard Scanner](https://img.shields.io/badge/HOL%20Guard-passing-00a67e)](https://github.com/hashgraph-online/hol-guard)
[![Plugin Security Scan](https://github.com/poliakarmai/gsc/actions/workflows/plugin-scan.yml/badge.svg)](https://github.com/poliakarmai/gsc/actions/workflows/plugin-scan.yml)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

## English

**GSC is a self-learning AppSec platform that proves exploits, verifies fixes, and heals CI.**

```
detect → prove → fix → verify → heal → learn
```

Most scanners report matches. GSC **proves them**: every finding can be turned into a
runnable proof-of-concept, validated in a sandbox, then fixed and re-verified by
re-running the exploit.

**What makes it different:**

- **69 detectors** across SAST, SCA, secrets and IaC — plus a YAML rule DSL.
- **PoC auto-generation** with a safe-exploitation gate: a finding ships with a working exploit, not just a line number.
- **Proof-of-Fix** — a patch is only accepted after the exploit stops firing in a sandbox.
- **Self-learning** — detectors are re-scored nightly against a calibration network; noisy patterns are auto-deactivated at <30% TP. Federated feedback is privacy-first (`{tenant_hash, rule_id, tp, fp}` + DP noise).
- **Priority by exploitability, not CVSS** — EPSS + CISA KEV + ExploitDB.
- **BYO-LLM** — the engine, detectors and PoC logic are ours; LLM revalidation runs on *your* key (any OpenAI-compatible endpoint, including a local Ollama). No key → fast regex-only mode.

**Use it as a GitHub Action** (see below) or as an **MCP server** with six read-only tools —
`scan_repo`, `scan_diff`, `list_findings`, `get_finding`, `verify_finding`, `list_detectors` —
for Claude Code, Codex, Cursor and other MCP clients. Destructive actions stay out of the
MCP surface by design.

**Proven on real code:** [HALL_OF_FAME.md](HALL_OF_FAME.md) ·
first public case study: [CASE_STUDY_CYBERBRO.md](CASE_STUDY_CYBERBRO.md)

### Measured on 100 real GitHub projects

The same fixed corpus is re-scanned after detector changes — 100 real repositories
(90 clean + 10 known-vulnerable), pinned to specific commits, **regex-only, no LLM key**.
Same projects every run, so a delta is the fix and not a new dataset.

| | 2026-08-21 (baseline) | 2026-08-28 (after FP fixes) |
|---|---|---|
| Findings | 64,831 | 41,127 |
| CRITICAL | 4,302 | 414 |
| HIGH | 37,246 | 2,187 |
| Known-vulnerable projects flagged | 8 / 10 | **10 / 10** |
| Clean projects with ≥1 false CRITICAL | 48 / 90 | 41 / 90 |

CRITICAL precision on a labeled review of 434 findings: **~4–5% at baseline → 31.1%**
(TP = 135, FP = 299, 2026-08-26). That is still noisy. The remaining false-positive mass is
concentrated in GS001 (secrets), GS005 (SQLi), GS020 (XSS) and GS037 (path traversal) and is
worked down rule by rule — we publish the numbers we have, not the numbers we would like.

*Method note:* corpus, severity counting and per-rule aggregation are ours; the engine ships as
a closed image, so reproducing this means running `gsc scan --ci --json` over the same pinned
projects and aggregating the same way. No clean-room comparison against other vendors is implied.

**License:** this repo (Action, docs, marketing) is **Apache-2.0**. The engine ships as a
closed Docker image `ghcr.io/poliakarmai/gsc-scanner` under **BSL 1.1** (free to scan your own
repos; competing SaaS requires a commercial license; converts to Apache-2.0 on 2030-08-06).

### Case study: XSS in a 122★ OSINT platform (stars as of 2026-08-08)

First public proof — [`stanfrbd/cyberbro`](https://github.com/stanfrbd/cyberbro) (46,677 LOC, 281 files):

| | |
|---|---|
| Finding | **CWE-79** XSS via search highlight — `static/history.js:148`, `p.innerHTML = p.title.replace(...)` |
| Severity | HIGH (CVSS 7.5) |
| Detector | **GS020** (`gs020_xss_injection`) |
| Fix | 9 lines, 1 file — exploit re-run in sandbox confirms it no longer fires |

Full write-up → [CASE_STUDY_CYBERBRO.md](CASE_STUDY_CYBERBRO.md) ·
more findings in production OSS → [HALL_OF_FAME.md](HALL_OF_FAME.md)

---

**GSC — самообучающаяся AppSec-платформа полного цикла.**

```
detect → prove → fix → verify → heal → learn
```

## 🚀 Быстрый старт — GitHub Action

Проверьте свой репозиторий за 30 секунд. Добавьте файл `.github/workflows/gsc.yml`:

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
      - uses: poliakarmai/gsc@v1   # для воспроизводимости — pin на commit SHA
        with:
          fail_on_critical: false   # true = блокировать merge при CRITICAL
```

Откройте pull request — GSC отсканирует код, пришлёт комментарий с находками
(CRITICAL / HIGH) и поставит оценку безопасности. Ядро поставляется как
закрытый Docker-образ `ghcr.io/poliakarmai/gsc-scanner` — исходный код движка
не публикуется.

> **Воспроизводимость.** `@v1` и тег образа — плавающие ссылки: результат может
> измениться без изменения вашего workflow. Для аудита/CI фиксируйте
> `poliakarmai/gsc@<commit-sha>` и образ по digest
> (`ghcr.io/poliakarmai/gsc-scanner@sha256:<digest>`).

**Входные параметры:**

| Вход | По умолчанию | Что делает |
| --- | --- | --- |
| `path` | `.` | Что сканировать (корень репозитория) |
| `deep_scan` | `true` | Глубокий скан (Echelon 4, медленнее) |
| `fail_on_critical` | `false` | `true` — упасть при CRITICAL |
| `fail_on_score` | `0` | Упасть, если score ниже порога. `0` = выключено, диапазон 1–100 |
| `max_findings_to_comment` | `10` | Сколько строк показывать в комментарии на каждую секцию |
| `llm_api_key` | — | BYO-LLM ключ для ревалидации (без него — regex-only) |
| `llm_base_url` | — | Свой OpenAI-совместимый эндпоинт (DeepSeek/OpenRouter/Ollama) |
| `llm_model` | — | Имя модели для ревалидации |
| `sarif` | `false` | Дополнительно отдать SARIF для GitHub Code Scanning |
| `reachability` | `false` | Понизить находки в недостижимых файлах (import-graph) |

**Выходы:** `score` (0–100) и `grade` (A+ … F).

> Комментарий в PR считает находки **по локациям** (`file:line`), а не по
> срабатываниям детекторов: один секрет, пойманый четырьмя правилами, — это
> одна строка с перечислением правил, а не четыре. `Total` в таблице всегда
> равен сумме категорий.


LLM-ревалидация — на вашем ключе (BYO-LLM):
```yaml
- uses: poliakarmai/gsc@v1
  with:
    llm_api_key: ${{ secrets.DEEPSEEK_API_KEY }}   # DeepSeek / OpenRouter / GMI / Ollama
```
Без ключа — быстрый regex-only скан (уязвимости находит; confirmed-вердикты — с ключом).

## Что умеет

| Слой | Возможности |
|---|---|
| **SAST** | 69 детекторов (SAST · SCA · Secrets · IaC), YAML rule DSL, LLM-триаж с confidence scoring |
| **Доказательство** | PoC auto-generation — рабочий эксплойт для находки *(движок; CLI/engine, не MCP)* |
| **Исправление** | Proof-of-Fix — патч верифицируется перезапуском эксплойта в sandbox; self-healing CI (авто-PR) *(CLI, требует явного флага)* |
| **Приоритизация** | EPSS + CISA KEV + ExploitDB (не сырой CVSS), БДУ ФСТЭК |
| **Recon** | passive reconnaissance: subdomain / tech / DNS / HTTP |
| **Supply chain** | SBOM (CycloneDX / SPDX) + VEX |
| **Compliance** | маппинг находок на ГОСТ Р 56939-2024, PCI DSS 4.0, SOC2, ISO 27001, CWE, OWASP — отчёт для ФСТЭК / КИИ / аудита |
| **Интеллект** | security archaeology, predictive forecasting, NL-policy, federated self-learning |

> **Где что доступно.** Возможности движка не равны возможностям Action/MCP:
>
> | Возможность | Action | MCP | CLI / engine image |
> |---|---|---|---|
> | Скан (SAST/SCA/Secrets/IaC) | ✅ | ✅ | ✅ |
> | PoC auto-generation | — | — | ✅ (`--with-poc`) |
> | Proof-of-Fix | — | — | ✅ (`pof generate`) |
> | Self-healing CI (авто-PR) | — | — | ✅ (`--create-pr`, human-approval) |
>
> В MCP **нет** деструктивных действий (auto-patch/PR) — осознанно, они остаются
> в CLI/human-лупе. `verify_finding` перезапускает PoC в sandbox, то есть по
> побочным эффектам это не «чисто read-only».

### Что уже внутри (код, а не план)

| Механика | Где | Что делает |
|---|---|---|
| **Мутационный re-attack PoF** | `gsc_cli/gsc_poc_mutator.py` → `gsc_proofoffix.py` | после фикса PoC перезапускается в мутированных формах (url-encode, double-encode, HTML-entities, case-swap, tab/space) — ловит «залатали симптом, а не причину». Детерминированно, без LLM |
| **Positive control + taint-recheck** | `gsc_pof_sandbox.py`, `tests/test_taint_recheck.py` | PoC обязан срабатывать на уязвимом варианте и молчать на исправленном |
| **Контейнерная изоляция hostile-PoC** | `gsc_pof_sandbox.py::_isolation_backend()` | docker/podman (`--network none`, read-only, cap-drop, non-root) + rlimit-fallback; уровень изоляции возвращается в каждом результате |
| **Quick-режим без LLM** | `gsc external-scan --scan-mode quick` | regex-only: находки без LLM-вердиктов, 0 токенов |

## 🎯 Наша фишка: BYO-LLM — движок наш, судья твой

GSC — это **движок + детекторы + PoC**, а не очередная обёртка над LLM.

- **От GSC:** 69 детекторов, V3-scoring, FP-фильтр, PoC/PoF, self-learning — закрытый IP в Docker-образе.
- **От вас:** LLM-ключ (любой OpenAI-совместимый — DeepSeek/OpenRouter/GMI/локальный Ollama) **или** ваш AI-агент (Claude/Cursor) через MCP.

Почему это выгодно вам:
- **Ноль расходов на чужие токены** — ревалидация на вашем ключе.
- **Никакой привязки к провайдеру** — хоть бесплатный локальный Ollama.
- **Вы платите за детекторы и логику**, а не за вызовы LLM.

## 🧠 Второй столп: Self-Learning — детекторы обновляются сами, каждую ночь

Помимо BYO-LLM у GSC есть **замкнутый контур обучения без LLM** (0 токенов, детерминированный):

```
скан → авто-разметка findings по calibration-сети (16 проектов: 9 clean + 7 vuln)
     → авто-деактивация шумных паттернов (<30% TP)
     → пополнение эталонов из security-фидов
     → federated feedback (приватно: только {tenant_hash, rule_id, tp, fp} + DP-шум)
```

Детекторы учатся на **реальных находках клиентов** через federated-контур — то,
чего мейнстрим сознательно избегает (риск дрейфа). GSC обходит это консервативным
self-learning: разметка по фиксированной сети + дискретные деактивации, без
переписывания весов — скорость ежедневного апдейта без катастрофического дрейфа.

| Вендор | Как обновляет правила | Self-learning |
|---|---|---|
| Semgrep | вручную (`semgrep-rules`) | ❌ |
| GitHub CodeQL | вручную (Security Lab) | ❌ |
| Snyk Code | офлайн-ML, «never customer data» | ❌ |
| SonarQube | вручную (SonarSource) | ❌ |
| **GSC** | **ежедневно · 0-LLM · federated** | ✅ |

**Никто не замыкает петлю обучения в проде — GSC закрывает этот зазор.**
Подробно, с пруф-базой (доки вендоров + research-статьи) →
[SELF_LEARNING.md](SELF_LEARNING.md).

> **Честная рамка.** GSC не заменяет Semgrep/CodeQL: у них шире покрытие языков и
> зрелее правила. GSC закрывает то, чего они не делают — доказывает находку PoC,
> верифицирует фикс перезапуском эксплойта, учится на TP/FP. Рабочая схема — вместе:
> зрелый сканер на массовый охват + GSC на закрытие конкретных находок.

## 🔌 Интеграции

### Paperclip — «найми GSC как сотрудника по безопасности»

[Paperclip](https://github.com/paperclipai/paperclip) (79K★, open-source оркестратор AI-агентов)
подключает GSC через MCP Tool Gateway — любой нанятый агент (Claude Code / Codex / Hermes)
получает GSC-инструменты и роль «Security Engineer».

- **6 MCP-инструментов:** `scan_repo` · `scan_diff` · `list_findings` · `verify_finding` · `get_finding` · `list_detectors`
- **Гайд + skill `gsc-security-review`** — в `docs/integrations/paperclip/` (репозиторий gsc-core)
- **Транспорт:** `local_stdio` (subprocess) или `mcp_remote` (HTTP)

Подключение за минуты: зарегистрируй `ToolStdioCommandTemplate`, создай gateway — и GSC сканирует репозитории твоей Paperclip-компании.

## Лицензия и доступ

**Этот репозиторий (GitHub Action, документация, маркетинг) — Apache License 2.0.** Свободно используйте Action в своих репозиториях.

Движок GSC (детекторы, PoC/PoF, калибровочная сеть, LLM-слой) поставляется отдельно как закрытый Docker-образ `ghcr.io/poliakarmai/gsc-scanner` под **Business Source License 1.1 (BSL 1.1)**:

- **Свободное использование** — включая сканирование собственных репозиториев в production — разрешено всегда.
- **Коммерческий SaaS/SAST-сервис** для третьих лиц, конкурирующий с GSC, — требует коммерческой лицензии (после NDA).
- **Change Date 2030-08-06** → движок автоматически переходит на Apache 2.0.

Для демо, пилота или приобретения — свяжитесь с автором.

© 2026 Алексей Поляков.
