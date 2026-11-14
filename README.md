# Paypilot-Prykhodchenko

Домашнє завдання №1: Quality Bar Proposal для PayPilot (стадія Seed, мандат Ship it) — [`quality-bar-proposal.md`](quality-bar-proposal.md).

## Прогін

- Скрипт: `l02_eval.py` (DeepEval 4.2.6, `requirements.txt`), кейси: `cases.json` (13 кейсів), `cases-step5.json` (C-03, C-04, C-08 з мінімальним курованим контекстом для hallucination rate).
- Профілі: `clean` і `lesson-02`.
- Агент і суддя: `claude-haiku-5-5` (`JUDGE_MODEL` за замовчуванням).
- `CLOCK_OVERRIDE=2026-09-15T10:00:00Z` — скрипт ставить годинник стенду сам.
- Сирі звіти: `reports/*.json`.

## Як відтворити

Потрібні Docker з compose і локальний стенд PayPilot на `http://localhost:8000` з провайдером `anthropic` і моделлю `claude-haiku-5-5`.

```bash
cp .env.example .env    # вписати ANTHROPIC_API_KEY і STAND_DIR — шлях до каталогу стенду
docker compose build
```

| Звіт | Команда |
|---|---|
| `l02-clean-20261111-213843.json`, `l02-clean-20261111-214142.json` (baseline) | `docker compose run --rm eval --profiles clean` — двічі |
| `l02-clean-lesson-02-20261111-215852.json` | `docker compose run --rm eval --runs 3` |
| `l02-clean-lesson-02-20261111-212504.json` | `docker compose run --rm eval --runs 3 --baseline-runs 2` |
| `l02-clean-lesson-02-20261112-222148.json` | `docker compose run --rm eval --cases cases-step5.json --metrics domain,faithfulness,hallucination --runs 3 --baseline-runs 1` |

Baseline можна зняти й одним запуском: `docker compose run --rm eval --profiles clean --baseline-runs 2`.
