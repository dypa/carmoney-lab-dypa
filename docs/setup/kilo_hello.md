# kilo_hello

1) Учебный сервис предварительной оценки заявки на заём под ПТС (carmoney-lab, практикум М3): принимает заявку (VIN, год, пробег, стоимость, сумма, срок), считает LTV и возвращает `approve` / `review` / `reject`; все данные синтетические.
2) В Makefile цели `help`, `up` (`docker compose up -d --build`), `down`, `ps`, `logs`, `install` (`composer install`), `test` (PHPUnit локально или в контейнере), `lint` (`php -l` по `backend/` и `tests/`), `seed` (залить `db/seed.sql` в MySQL); в `docker-compose.yml` сервисы `backend` (PHP на 8080) и `db` (mysql:8.0), переменные `APP_PORT` / `DB_PORT`.
3) Решение `approve` / `review` / `reject` считается в `backend/src/Domain/` (там лежат `DecisionEngine.php`, `LtvCalculator.php`, `AssessmentService.php` и валидаторы), пороги — в `backend/config/rules.php`.

модель: training-2026-09-minimax-m3 (MiniMax-M3, Kilo)