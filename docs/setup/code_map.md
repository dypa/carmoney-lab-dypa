# Карта кода: как считается решение approve / review / reject

Объект разбора — `backend/src/Domain/` и `backend/config/rules.php`. Точка входа
(`backend/src/Http/ApplicationController.php`, сборка в `backend/src/AppFactory.php`)
приведена, потому что без неё не виден порядок вызовов.

## Кто участвует

| Файл | Роль |
|---|---|
| `backend/config/rules.php` | справочник порогов и лимитов; код читает числа отсюда |
| `backend/src/Domain/ApplicationValidator.php` | валидация и нормализация заявки |
| `backend/src/Domain/VinValidator.php` | формат VIN (длина, алфавит, запрещённые символы) |
| `backend/src/Domain/VehicleAge.php` | возраст авто: `currentYear - productionYear` |
| `backend/src/Domain/LtvCalculator.php` | LTV в процентах |
| `backend/src/Domain/DecisionEngine.php` | решение по LTV — **единственное место, где рождается decision** |
| `backend/src/Domain/AssessmentService.php` | оркестратор: валидация → LTV → решение → лимит |
| `backend/src/Domain/ValidationException.php` | ошибки валидации → HTTP 422 |

Сборка: `AppFactory::create()` (`backend/src/AppFactory.php:25–39`) загружает `rules.php`
и создаёт все классы. `DecisionEngine` получает в конструктор **только** `rules['ltv']`
(`AppFactory.php:37`); весь справочник `rules` уходит в `ApplicationValidator`
(`AppFactory.php:31`).

## Порядок вызовов

Вход: `POST /api/ltv` → `ApplicationController::ltv()`
(`backend/src/Http/ApplicationController.php:54`), `POST /api/applications` → `create()`
(`:23`). Оба вызывают `AssessmentService::assess($payload)` (`:28`, `:59`).

`AssessmentService::assess()` (`AssessmentService.php:28`) — шаги по порядку:

1. **`ApplicationValidator::validate($payload)`** (`AssessmentService.php:30` →
   `ApplicationValidator.php:24`):
   - `VinValidator::isValid()` (`:29` → `VinValidator.php:18`): длина 17, только `A-Z` и
     `0-9`, без `I`/`O`/`Q` (пороги — `rules['vin']`); контрольная сумма VIN не считается
     (`VinValidator.php:9`);
   - год: не раньше `min_year` 1990, возраст не в будущем и не больше `max_age_years` 20
     (возраст через `VehicleAge::inYears()`, `ApplicationValidator.php:33–41`);
   - пробег: целое от 0 до `max_mileage_km` 500000 (`ApplicationValidator.php:43–46`);
   - `market_value` > 0 (`:48–51`); `requested_amount` 50000…2000000 (`rules['amount']`,
     `:53–60`); `term_months` 3…48 (`rules['term']`, `:62–69`);
   - любые ошибки → `ValidationException` (`:71–73`) → контроллер отвечает 422
     (`ApplicationController.php:29–31`).
2. **`LtvCalculator::calculate(requested_amount, market_value)`** (`AssessmentService.php:32`
   → `LtvCalculator.php:15`): `round(amount / value * 100, 2)`; при `value <= 0` или
   `amount <= 0` — `InvalidArgumentException` (после валидации недостижимо, но код есть).
3. **`DecisionEngine::decide($ltv)`** (`AssessmentService.php:33` → `DecisionEngine.php:30`):
   - `ltv < approve_max` (60.0) → `approve`;
   - `ltv <= review_max` (85.0) → `review`;
   - иначе `reject`.

   Границы по коду: `[0; 60)` — approve, `[60; 85]` — review, `(85; ∞)` — reject.
   Расхождение с комментариями: и `rules.php:39`, и `DecisionEngine.php:10` пишут
   `LTV <= approve_max -> approve`, а в коде строгое `<` (`DecisionEngine.php:32`) —
   при LTV ровно 60.0 код вернёт `review`.
4. **Результат** (`AssessmentService.php:35–41`): `vehicle_age` (ещё раз
   `VehicleAge::inYears()`), `ltv`, `decision`, `approved_limit` = `requested_amount`
   при approve, иначе 0. Лимит по возрасту из `rules['ltv_by_age']` не считается —
   задача LOAN-12 не сделана, справочник заполнен, но не подключён (`rules.php:48–58`,
   `AssessmentService.php:10–12`).

## Что сейчас проверяется про пробег

- **Только валидация**: `ApplicationValidator.php:43–46` — приводится к int (по
  умолчанию -1), ошибка, если меньше 0 или больше `rules['vehicle']['max_mileage_km']`
  (500000). Это ошибка валидации с ответом 422, а **не** решение `review`.
- В LTV, в решении и в лимите пробег **не участвует**: `LtvCalculator` его не получает,
  `DecisionEngine::decide(float $ltv)` видит только LTV. Других проверок пробега нет.
- Нормализованный `mileage` возвращается валидатором в `$input`
  (`ApplicationValidator.php:78`), попадает в ответ `assess()` под ключом `input` и
  передаётся в `ApplicationRepository::save()` вместе с остальными полями
  (`ApplicationController.php:34`).

## Куда встаёт правило «пробег ≤ 400 000 км, иначе review»

Решение рождается в одном месте — `DecisionEngine::decide()` (`DecisionEngine.php:30–41`),
но сигнатура `decide(float $ltv)` о пробеге не знает. Совместимые с текущей структурой
варианты:

- **Вариант A** — расширить `DecisionEngine`: передать пробег (`decide(float $ltv, int $mileage)`)
  и порог в конструктор (сейчас туда идёт только `rules['ltv']`, сборка —
  `AppFactory.php:37`); внутри — после LTV-веток (`DecisionEngine.php:32–40`) применить
  правило: пробег выше порога → `REVIEW`. Константа `DecisionEngine::REVIEW` уже есть
  (`DecisionEngine.php:17`).
- **Вариант B** — без правки движка: в `AssessmentService::assess()` после строки 33,
  где `$input['mileage']` уже доступен, скорректировать `$decision`.

Порог 400 000 — в `backend/config/rules.php` (конвенция `AGENTS.md`: бизнес-числа только
там), логичное место — секция `'vehicle'`.

### Входные данные: что уже есть

- пробег как провалидированное целое: `$input['mileage']`
  (`ApplicationValidator.php:43`, `:78`), в `assess()` доступен с строки 30;
- константа `DecisionEngine::REVIEW`;
- секция `'vehicle'` в `rules.php`, куда ляжет порог.

### Чего не хватает

- значения 400 000 в `rules.php` — **нет** (ближайшее `max_mileage_km` = 500 000 — это
  другая, валидационная граница, а не порог решения);
- доступа `DecisionEngine` к пробегу и к правилам `vehicle` — **нет** (конструктор получает
  только `rules['ltv']`);
- правила старшинства: если LTV уже дал `reject`, а пробег > 400 000 — понижать reject до
  review или правило действует только на approve? В коде **нет**, нужно решение человека
  (по `AGENTS.md` пороги меняются только решением человека);
- для варианта A — правки сигнатуры `decide()` и сборки в `AppFactory`.
