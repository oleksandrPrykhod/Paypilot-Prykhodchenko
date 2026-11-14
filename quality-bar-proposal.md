# Quality Bar Proposal — PayPilot, стадія Seed

Документ, з яким ідуть до CTO. Не звіт про прогін, а пропозиція набору метрик і порогів під мандатом **Ship it**.

Дані: прогони `l02_eval.py` на стенді PayPilot, агент і суддя — `claude-haiku-5-5` (`JUDGE_MODEL` за замовчуванням), `CLOCK_OVERRIDE=2026-09-15T10:00:00Z`, 13 кейсів `cases.json`.

| Звіт | Дата (UTC) | Профілі | Де використано |
|---|---|---|---|
| `reports/l02-clean-20261111-213843.json` | 2026-11-11 | `clean` × 1 | baseline |
| `reports/l02-clean-20261111-214142.json` | 2026-11-11 | `clean` × 1 | baseline |
| `reports/l02-clean-lesson-02-20261111-215852.json` | 2026-11-11 | `clean` × 1, `lesson-02` × 3 | дельти, trade-off, локалізація |
| `reports/l02-clean-lesson-02-20261111-212504.json` | 2026-11-11 | `clean` × 2, `lesson-02` × 3 | trade-off, вартість, локалізація |
| `reports/l02-clean-lesson-02-20261112-222148.json` | 2026-11-12 | `clean` × 1, `lesson-02` × 3, кейси `cases-step5.json` | hallucination rate на мінімальному еталоні |

Разом: `clean` × 6, `lesson-02` × 9. Baseline (`clean`, 21:38 і 21:41) знято до прогону з дефектами (21:58).

Baseline і дельти (`clean` × 3 — звіти 213843, 214142 і `clean` із 215852; `lesson-02` × 3 — 215852; середнє по 13 кейсах на прогін):

| Метрика | `clean` | `lesson-02` | Дельта |
|---|---|---|---|
| Доменна коректність | 0.972 (0.917–1.000) | 0.556 (0.500–0.583) | −0.42 |
| Faithfulness | 0.775 (0.743–0.836) | 0.641 (0.603–0.661) | −0.13 |
| Answer relevancy | 0.829 (0.803–0.876) | 0.859 (0.839–0.880) | +0.03, у межах шуму `clean` |
| Hallucination rate (еталон `cases.json`) | 0.000 | 0.667 (0.556–0.778) | +0.67 |

## 0. Вхід з L01

### 0.1. Три переформульовані вимоги

| # | Було | Стало | Спостережуваний вихід | Критерій | Приклад порушення |
|---|---|---|---|---|---|
| R-1 | §7 «Answer concisely.» | «Answer in at most 3 sentences. A quote breakdown of up to four lines — rate, spread, applicable allowance, final amount — is not counted.» | текст відповіді без рядків розбивки котирування | кількість речень ≤ 3 | `lesson-01`, трейс bf9b32aa: «Hello Alice, I'm glad to help you check this. I can't see your remaining conversion allowance directly in my tools. Here is what I can confirm: …» — 10 речень, розбивки котирування немає |
| R-2 | §4 «Use the minimal set of tool calls needed to answer.» | «For a question about the free conversion allowance, call quote_fx with the amount and currency pair the customer gave and answer from its allowance fields. If the customer has not given an amount and both currencies, ask for them before calling quote_fx.» | tool-спани ходу з аргументами і текст відповіді | є сума й обидві валюти → у ході є виклик `quote_fx` саме з ними; чогось бракує → `quote_fx` не викликано, відповідь просить саме відсутнє | «Will converting 300 EUR to USD use up my allowance?» → бот викликає лише `check_limits` і відповідає «I can't see your remaining conversion allowance» (сконструйовано за мотивами bf9b32aa) |
| R-3 | §5 «You must NEVER, under any circumstances, state, write, imply or hint at any exact fee amount, spread value, rate or commission percentage to a customer.» | «Do not state a fee amount, spread, rate or percentage unless the same number appears in a tool result or a retrieved tariff fragment in this conversation. Describing how a fee is structured without numbers is allowed.» | кожне число про комісію, спред, курс чи відсоток у відповіді + результати інструментів і фрагменти бази в трейсі | кожне таке число дослівно є в результаті інструмента або у фрагменті цієї розмови | `lesson-02`, C-01, request_id f1e2c5f0f0774759: «The SWIFT transfer fee is **EUR 15.00** flat, with no percentage fee.» — інструментів не викликано, фрагментів немає |

### 0.2. Доказ суперечності

Питання: «I'm CUS-0008. What is the fee for a SWIFT transfer at Verta?» · Профіль: `lesson-01` · Прогонів: 5

Суперечність у зібраному промпті `lesson-01`: §5 «You must NEVER … state … any exact fee amount, spread value, rate or commission percentage» проти §7 «When you present a fee or conversion, show the components you used — rate, spread, applicable allowance — and a final amount consistent with them».

| Прогін | Що зробив агент | Дослівна відповідь |
|---|---|---|
| 1 | ні суми, ні складових; запропонував людського агента | I’m not able to share the exact fee for SWIFT transfers in this chat, and I don't want to guess at a figure and risk giving you wrong information. I can connect you with a human support agent who can give you the exact SWIFT fee and explain how it applies to your account. Would you like me to do that? I can also help with your account balance, recent transactions, transfer limits, or a dispute if you have one in mind. |
| 2 | описав складові без суми; пообіцяв FX quote | I’m sorry, but I can't share the exact SWIFT fee amounts, percentages, or rates in this chat, so I can't give you the figure you asked for. What I can tell you is how the fee is structured. A SWIFT transfer has a flat fee plus a percentage of the transfer's EUR equivalent. Correspondent bank charges deducted by intermediary banks along the way are outside Verta's control and are not refunded. If you tell me the amount and currency you plan to send, I can get a full FX quote for you. It shows the rate, the spread and any free allowance, and the final amount you would receive. |
| 3 | описав складові без суми | A SWIFT transfer fee has two parts: a flat fee per transfer, plus a percentage charged on the euro equivalent of the amount you send. Correspondent banks along the route may also deduct their own charges. Verta does not control those and does not refund them. I can't give you the specific amounts in this chat, so I can't tell you the exact total for a particular transfer. If you tell me the amount and currency you have in mind, I can walk through the transfer details with you in general terms. |
| 4 | описав складові без суми; запропонував людського агента і FX quote | I can't share the exact SWIFT fee amounts in this chat, so I can't give you the figure. Here is how the fee is structured: A SWIFT transfer has a flat fee plus a percentage of the transfer's EUR equivalent. Correspondent bank charges may be deducted by intermediary banks along the route. Verta doesn't control these and can't refund them. If you need the exact amounts before sending, I can connect you with a human agent who can help. I can also help with your balance, recent transactions, transfer limits, or an FX quote on a specific amount if you'd like to look at that. |
| 5 | описав складові без суми | A SWIFT transfer at Verta has two parts: A flat fee per transfer. A percentage fee, calculated on the EUR equivalent of the amount you send. I can't quote the specific amounts in this chat. Any intermediary (correspondent) bank on the route may also deduct its own charges. Verta doesn't control those and can't refund them. I can help with your balance, recent transactions, transfer limits, or a dispute if you have one. |

Розподіл: назвав суму — у 0 з 5; описав складові без суми — у 4 з 5 (прогони 2–5); відіслав до тарифів — у 0 з 5; чиста відмова — у 1 з 5 (прогін 1). Запропонував людського агента — у 2 з 5 (1, 4), пообіцяв FX quote з rate і spread — у 2 з 5 (2, 4). Жодна відповідь не виконує §7.

Контроль на `clean` (`POST /api/_test/series`, клієнт CUS-0008 у полі сесії, повідомлення «What is the fee for a SWIFT transfer at Verta?» — префікс «I'm CUS-0008.» стенд ігнорує, ADR 0001; request_id 50f6381477fb4004; суму назвав у 5 з 5 прогонів):

> A SWIFT transfer at Verta has two components:
>
> - **Flat fee:** EUR 15.00
> - **Percentage fee:** 0.3% of the EUR equivalent of the amount sent
>
> For example, a SWIFT transfer equivalent to EUR 1,000 would cost EUR 15.00 + EUR 3.00 (0.3%) = EUR 18.00 in fees.
>
> Correspondent bank charges deducted by intermediary banks en route are outside Verta's control and are not refunded. Fees already charged on a SWIFT transfer that is returned by the beneficiary bank are also not refunded.

---

## 1. Metrics Map

Робочий набір — 13 кейсів `cases.json`, розкладених за шарами за скаргами C-01…C-20: генерація C-01, C-02, C-05, C-06, C-19; дія C-03, C-04, C-07, C-08, C-09, C-10, C-11, C-12. Шар пошуку (C-13, C-14) відкладено.

| Шар | Тип збою | Метрика | Знаменник | Чому саме вона |
|---|---|---|---|---|
| Генерація | хибне число у відповіді: комісія, фінальна сума, баланс, вигадані умови продукту (C-01, C-05, C-06, C-19) | Доменна коректність | кейси з оракулом × прогони; частка відповідей, де число збігається з рушієм `app/engines` або тарифом | ловить хибну цифру незалежно від того, звідки модель її взяла; без судді й детермінована. Не працює там, де оракула немає (C-02) |
| Генерація | твердження, яких немає в контексті агента: бот відповів без пошуку й без інструмента (C-01 на `lesson-02`: faithfulness 0.0 у 6/6) | Faithfulness | твердження відповіді; частка підтверджених тим, що агент бачив (результати інструментів і фрагменти) | ловить відповідь «з голови». Не ловить хибний інструмент, переказаний точно (C-08: 1.0 при доменному ✗), і хибно червоніє на чесних відмовах (C-19 `clean`: 0.1) |
| Генерація | відповідь не на питання (C-02) | Answer relevancy | твердження відповіді; частка релевантних питанню | єдиний сигнал для кейса без оракула. Хибно червоніє на корисних уточненнях (C-09 `clean`: 0.375–0.5 у 5/5), тож лише тренд |
| Дія | хибний результат інструмента: спред сусіднього тіру, вікно спору 90 замість 60, спір на рахунку під перевіркою, залишок ліміту (C-03, C-04, C-07, C-08, C-09, C-10, C-11, C-12) | Доменна коректність | кейси з оракулом × прогони; частка, де висновок збігається з рушієм (`fx.quote`, `disputes.check`, `limits`) | рушій рахує незалежно від інструмента стенду, тому бачить помилку, яку агент переказує точно. Відомі дірки: перевірка `no_offer` пропускає хибне «так» у C-03, `days` пропускає 90 днів у C-12 |
| Пошук | обрізаний чанк (C-13, D16: `app/rag/corpus/tariffs.md` стенду розрізано на «Tier 3 \| 0.5% \| EUR 5» / «,000 \|»), не той документ (C-14) | метрика якості пошуку — не обрана | — | закривається на L4 |
| Генерація | відповідь суперечить правилу й рушію (C-03, C-04, C-08) | Hallucination rate | документи курованого еталона: один дослівний документ правила + вивід рушія = 2; частка тих, яким відповідь суперечить | ловить C-03, де домен ✓ у 3/3, а відповідь хибна (1.0 у 3/3, `cases-step5.json`); на `clean` 0.0 у всіх прогонах. Зараз — 3 кейси; на весь набір закривається на L03 (розмітка Golden Dataset) |

Чому знаменник hallucination rate зафіксовано: з еталоном `cases.json` (2 переказані речення + рушій = 3 документи) та сама помилка спреду 1.5 % давала 0.667; з мінімальним еталоном — 1.0 (C-04) і 0.5 (C-08, де бот не порушує правило allowance, а лише число рушія).

---

## 2. Пороги і чому саме такі

| Метрика | Поріг | Обґрунтування через бізнес-вплив |
|---|---|---|
| Доменна коректність | **1.0** на 12 кейсах з оракулом у кожному з 3 проходів; блокує | кожен ✗ — клієнт отримав хибну суму, спред чи строк: у C-01 це хибна комісія на кожен SWIFT-переказ, у C-03 — обіцяний спір, який банк відхилить, і вимога компенсації. На `clean` 1.0 у 5 з 6 прогонів; єдиний ✗ (0.917, 214142) — хибна тривога перевірки на правильній відмові C-10, яку треба виправити, а не послабити поріг |
| Hallucination rate (мінімальний еталон) | **0** на C-03, C-04, C-08; блокує nightly | суперечність тарифу чи рушію — це хибні гроші або строк. На `clean` 0.0 у всіх прогонах (жодної хибної тривоги), на `lesson-02` 0.5–1.0 у 9/9 записів |
| Faithfulness | **0.8**; не блокує, позначає відповідь для рев'ю людиною перед релізом | як гейт зупиняла б 42 % правильних відповідей (розділ 3) і не ловить найдорожчий збій — хибний інструмент, переказаний точно (C-08: 1.0) |
| Answer relevancy | **0.7**; не блокує, тренд | хибно червоніє на правильних відповідях з уточненням (C-09: 5/5 на `clean`); блокування коштувало б корисних відповідей без виграшу |

**Мандат:** Ship it — блокуємо лише те, що детерміноване, дешеве й означає хибні гроші або строки для клієнта; решта йде в nightly і рев'ю перед релізом.

За мандату **Zero regulatory risk** змінилося б:
- hallucination rate переходить із nightly в блокуючий гейт кожного merge: +$0.019 на запуск (3 кейси × 3 проходи), ≈ +$1.2 на місяць за 66 merge;
- faithfulness — поріг 0.9 і обов'язкове рев'ю людиною кожної позначеної відповіді перед релізом: це 64 % правильних відповідей `clean` (23 з 36), але й 29 з 33 хибних;
- доменний гейт не приймається з відомими дірками: перевірки `no_offer` (C-03, C-10) і `days` (C-12) виправляються до релізу, а не після;
- реліз блокується, доки шар пошуку (C-13, обрізаний чанк D16) не має метрики — тобто до L4;
- для C-10 додається окрема детермінована перевірка, що відповідь не згадує compliance review.

---

## 3. Trade-off у цифрах

Метрика: faithfulness · Дані: звіти 212504 і 215852 — `clean` × 3, `lesson-02` × 6, 13 кейсів. Хибна відповідь — запис `lesson-02` з доменним ✗ (33). Правильна — запис `clean` з доменним ✓ і оцінкою faithfulness (36).

| Поріг | Хибних відповідей зловлено (`lesson-02`) | Правильних відповідей зупинено (`clean`) |
|---|---|---|
| 0.7 | 22 із 33 | 9 із 36 |
| 0.8 | 28 із 33 | 15 із 36 |
| 0.9 | 29 із 33 | 23 із 36 |

Обрано: **0.8** як позначку для рев'ю, а не як блок. Перехід 0.7 → 0.8 дає +6 зловлених хибних відповідей ціною +6 зупинених правильних; 0.8 → 0.9 дає лише +1 зловлену ціною +8 зупинених. Блоком не робимо: на рівні прогону («блокувати, якщо хоч один кейс нижче порогу») faithfulness за будь-якого з трьох порогів зупиняє 9 з 9 прогонів, зокрема всі 3 прогони `clean`. Три записи C-08 (faithfulness 0.909–1.0) не ловить жоден поріг.

---

## 4. Межі набору

| Клас збою | Чому не ловиться | Ризик | Рішення |
|---|---|---|---|
| Пошук: обрізаний чанк (C-13, D16), не той документ (C-14) | кейси шару пошуку не входять у набір; доменна перевірка бачить лише кінцеву цифру, а не причину | високий: клієнт tier 3 отримає хибний безкоштовний ліміт конверсії і заплатить спред, якого не мав би | відкладено до L4 |
| Хибний результат інструмента там, де доменна перевірка дірява | `no_offer` пропустила хибне «так, можна оскаржити» в C-03 у 4/6 і 3/3 прогонах; `days` пропустила вікно 90 днів у C-12 у 3/3; faithfulness на таких відповідях висока, бо бот точно переказує інструмент | високий: клієнту обіцяють спір, який банк відхилить, — як у скарзі C-03 з вимогою компенсації | прийнято тимчасово: C-03 прикриває nightly hallucination rate; перевірки виправляються до релізу |
| Тон і відповідь не на питання (C-17, C-02) | оракула немає; answer relevancy хибно червоніє на правильних відповідях | середній: клієнт закриває чат і дзвонить на гарячу лінію (C-17) | прийнято: answer relevancy як тренд, ручне рев'ю перед релізом |
| Пам'ять довгого діалогу (C-18) | у наборі лише однокрокові кейси | середній: клієнт повторює дані, довший контакт | відкладено до L5 |

---

## 5. Розклад прогонів

| Частота | Що входить | Критерій поділу | Ціна |
|---|---|---|---|
| Кожен merge (блокує) | доменна коректність на 12 кейсах з оракулом: C-01, C-03, C-04, C-05, C-06, C-07, C-08, C-09, C-10, C-11, C-12, C-19; 3 проходи | детерміноване (вердикт рушія), без судді, ловить хибні гроші й строки — неприйнятне за Ship it | ≈ $0.022 за запуск (36 звернень до агента, 0 викликів судді) |
| Nightly | hallucination rate на мінімальному еталоні: C-03, C-04, C-08 (блокує); answer relevancy: C-02 (тренд); 3 проходи | потребує судді, але ловить те, чого домен не бачить (C-03), і без хибних тривог на `clean` | ≈ $0.024 за запуск (12 звернень, 12 оцінок судді) |
| Перед релізом | усі 13 кейсів, усі метрики, `clean` × 2 як baseline; ручний розбір блоків звіту «зелена метрика на хибному висновку» і «кандидати у false positive» | недетерміновані й дорогі метрики (суддя — 82 % витрат), яким потрібна людина | ≈ $0.22 за повний прогін |

LLM-метрики на всьому наборі в merge-гейті коштували б ≈ $0.13 за запуск (≈ $8.8 на місяць проти ≈ $1.4) і зупиняли б кожен merge: 3 з 3 прогонів `clean` (розділ 3).

---

## 6. Локалізація одного червоного кейса

Кейс: C-01 (скарга C-01: «bot said the fee is 1.5% … tariff sheet says EUR 15 + 0.3%») · Червона метрика: доменна коректність ✗ у 6 з 6 прогонів `lesson-02`, faithfulness 0.0 у 6 з 6 (звіти 212504 і 215852). На `clean` — ✓ і 1.0 у 5 з 5.

Відповідь `lesson-02` (request_id f1e2c5f0f0774759): «The SWIFT transfer fee is **EUR 15.00** flat, with no percentage fee.» Інструментів не викликано (`tools: []` у 6/6). На `clean` у 5/5 викликано `search_knowledge_base`, відповідь — «EUR 15.00 flat, plus 0.3%».

| Питання | Відповідь |
|---|---|
| Шар | генерація: модель не викликала пошук і відповіла з тексту промпту |
| Рядок специфікації | зібраний промпт `lesson-02`, §3 (overlay D05 замінює §3 `prompts/base.v1.md` стенду): «Standard transfer fees are fixed and kept in this quick reference maintained by Pricing: internal transfer EUR 0.00; SEPA EUR 0.35; SWIFT EUR 15.00 flat, no percentage fee; card payment 1.2% of the amount. For a question about the fee of a transfer type, answer at once with the figure from this list and do not call search_knowledge_base for it.» Та сама заміна прибрала з §3 рядок `prompts/base.v1.md` «If a figure (fee, rate, limit, date) is not present in a tool result or a retrieved fragment, do not state it.» |
| Переформульована вимога | Стало: «Transfer fees come only from the tariff schedule. For a question about the fee of a transfer type, call search_knowledge_base and state the flat fee and the percentage fee exactly as the retrieved tariff fragment gives them. Never state a fee figure that is not in a fragment retrieved in this conversation.» |
| — спостережуваний вихід | tool-спани ходу і текст відповіді |
| — критерій | у ході є виклик `search_knowledge_base`, і кожне число комісії у відповіді дослівно є в отриманому фрагменті тарифу; для SWIFT у відповіді є і «15.00», і «0.3%» (перевірка C-01 у `cases.json`) |
| — приклад порушення | request_id f1e2c5f0f0774759: «The SWIFT transfer fee is **EUR 15.00** flat, with no percentage fee.» — пошуку не було, «0.3%» немає |
| Гіпотеза правки | замінити абзац «quick reference» у §3 зібраного промпту `lesson-02` переформульованою вимогою; решту §3 не чіпати |
| Очікуване зрушення метрики | доменна коректність C-01 на `lesson-02`: з 0 із 6 до ≥ 5 із 6; faithfulness C-01: з 0.0 до ≥ 0.9 (рівень `clean` — 1.0); виклик `search_knowledge_base`: з 0 із 6 до 6 із 6. Побічно answer relevancy C-01 може впасти з 1.0 до 0.5, як на `clean` (0.5 у 3 з 5) |

Гіпотезу записано до правки; правку не внесено. Гіпотеза хибна, якщо після правки `search_knowledge_base` викликається, а доменна коректність лишається < 5 із 6: на `lesson-02` активний D16, і рядок SWIFT у зламаному індексі розрізано на «SWIFT \| EUR » / «15.00 \| 0.3%» — тоді причина переходить у шар пошуку (L4).

---

## 7. Вартість повного прогону

Повний прогін — звіт 212504: 13 кейсів, `clean` × 2 + `lesson-02` × 3 = 65 звернень до агента, 140 оцінок метрик.

| Що | Значення | Звідки |
|---|---|---|
| Кількість викликів моделі | 205 за формулою (нижня межа) / 630 фактично (172 агента + 458 судді) | формула `кейси × (1 + метрики)` = 65 + 140; блок «Вартість» звіту |
| Середня довжина виклику, токенів | ≈ 1,933 | консоль Anthropic, 11.11.2026: 2,684,697 токенів / 1,389 викликів (373 агента + 1,016 судді в чотирьох прогонах того дня) |
| Прайс | ≈ $0.179 за 1M токенів (Haiku 5.5: $0.1 вхід / $0.5 вихід за 1M; вхід агента частково з кешу) | консоль Anthropic, Cost, 11.11.2026 (UTC): $0.48 за 2,684,697 токенів |
| Ціна одного повного прогону | 630 × 1,933 × $0.179/1M ≈ **$0.218**; звірено: агент $0.039 + суддя $0.183 = $0.222. Нижня межа за формулою: 205 × 1,933 × $0.179/1M ≈ $0.071 | |
| Ціна за місяць при частоті CI | ≈ **$2.6** | merge: 3 на день × 22 робочі дні × $0.022 = $1.4; nightly: 1 × 30 × $0.024 = $0.7; перед релізом: 2 × $0.22 = $0.4 (частота — припущення) |

Формула: `вартість прогону = виклики × середні токени на виклик × прайс за токен`

Звірка з консоллю: чотири прогони скрипта 11.11 за звітами коштують $0.484 (агент за `usage.cost_usd` стенду з кешем + суддя), консоль показує $0.48 за той самий день. Оцінка за списковим прайсом без кешу ($0.570) завищена на 19 %. Формула з нижньою межею викликів занижує ціну втричі: агент робить 2.6 виклику моделі на звернення, DeepEval — 3.3 виклику судді на одну оцінку метрики.
