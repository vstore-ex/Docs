# Декомпозиция PRD #18: Вертикальные срезы (Tracer Bullets)

Родительский PRD: [#18 feat(infra): HTTPS Edge Gateway & automated Let's Encrypt certificates via Certbot](https://github.com/vstore-ex/vstore-infra/issues/18)

---

### Список созданных задач (Vertical Slices)

| # | Задача | Блокируется | Статус | Ссылка |
|---|---|---|---|---|
| **1** | **feat(infra): Edge Gateway reverse proxy & HTTP-to-HTTPS redirect with ACME webroot** | *Нет (можно брать сразу)* | `ready-for-agent` | [#19](https://github.com/vstore-ex/vstore-infra/issues/19) |
| **2** | **feat(infra): Automated Certbot bootstrap script for initial certificate acquisition** | [#19](https://github.com/vstore-ex/vstore-infra/issues/19) | `ready-for-agent` | [#20](https://github.com/vstore-ex/vstore-infra/issues/20) |
| **3** | **feat(infra): On-boot and recurring certificate renewal service for intermittent server uptime** | [#19](https://github.com/vstore-ex/vstore-infra/issues/19), [#20](https://github.com/vstore-ex/vstore-infra/issues/20) | `ready-for-agent` | [#21](https://github.com/vstore-ex/vstore-infra/issues/21) |
| **4** | **feat(infra): CI/CD deployment integration, environment template & production runbook** | [#19](https://github.com/vstore-ex/vstore-infra/issues/19), [#20](https://github.com/vstore-ex/vstore-infra/issues/20), [#21](https://github.com/vstore-ex/vstore-infra/issues/21) | `ready-for-agent` | [#22](https://github.com/vstore-ex/vstore-infra/issues/22) |

---

### Детали срезов

#### Срез 1: [#19](https://github.com/vstore-ex/vstore-infra/issues/19) Edge Gateway reverse proxy & HTTP-to-HTTPS redirect with ACME webroot
* **Что строится:** Внедрение сервиса Edge Reverse Proxy на портах 80 и 443. 301 редирект HTTP -> HTTPS, пропуск `/.well-known/acme-challenge/`, SNI-маршрутизация на `frontend:8080` и `admin:8080`, снятие публичных биндингов 3000 и 3001 с хоста, генерация временных dummy-сертификатов для первичного старта.
* **Критерии приёмки:** Шлюз слушает 80/443; порты 3000/3001 закрыты; curl HTTP возвращает 301; curl ACME отдаёт токен; curl HTTPS с SNI отдаёт страницы фронтенда и админки; конфиг проходит валидацию compose.

#### Срез 2: [#20](https://github.com/vstore-ex/vstore-infra/issues/20) Automated Certbot bootstrap script for initial certificate acquisition
* **Что строится:** Скрипт инициализации (`init-letsencrypt.sh`) для чистого сервера. Проверяет наличие сертификатов, при отсутствии создаёт fallback-сертификаты, поднимает шлюз, запрашивает настоящие сертификаты Let's Encrypt через HTTP-01 webroot challenge (с поддержкой staging-флага) и делает релоад Nginx.
* **Критерии приёмки:** Автоматическое определение отсутствия сертификатов; корректная генерация боевых сертификатов; поддержка staging-режима; мягкий релоад шлюза; идемпотентность скрипта.

#### Срез 3: [#21](https://github.com/vstore-ex/vstore-infra/issues/21) On-boot and recurring certificate renewal service for intermittent server uptime
* **Что строится:** Фоновый сервис автообновления, адаптированный под сервер, работающий 1–2 часа в день. Запуск `certbot renew` немедленно при старте (On-Boot) + ежечасная проверка; безопасный релоад активной конфигурации Nginx без проброса `docker.sock`.
* **Критерии приёмки:** Запуск renew при каждом старте стека; ежечасный цикл проверки; быстрый выход (<2 сек), если обновление не требуется; автоматический подхват обновлённых сертификатов Nginx; изоляция контейнера.

#### Срез 4: [#22](https://github.com/vstore-ex/vstore-infra/issues/22) CI/CD deployment integration, environment template & production runbook
* **Что строится:** Интеграция с `.github/workflows/ci.yml`, обновление `.env.example` новыми переменными окружения, обновление ранбука в `vstore-infra/README.md` (настройка A-записей, первый запуск bootstrap, команды проверки curl).
* **Критерии приёмки:** Все переменные задокументированы в шаблоне; CI валидирует стек; автодеплой по SSH успешно накатывает изменения; ранбук содержит проверенные инструкции.
