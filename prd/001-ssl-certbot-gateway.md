# PRD: Внедрение защищённого HTTPS-шлюза (Edge Gateway) и автоматизации SSL-сертификатов Let's Encrypt через Certbot

- **Статус:** `ready-for-agent`
- **Область действия:** `vstore-infra` (Инфраструктура и оркестрация)
- **Целевая ветка:** `feat/ssl-certbot-gateway`

---

## Problem Statement

Клиенты интернет-магазина и сотрудники администрации в настоящее время вынуждены подключаться к сервисам по незащищённому протоколу HTTP на нестандартных портах (3000 для витрины и 3001 для панели управления).

Это влечёт за собой следующие критические проблемы:
1. **Предупреждения безопасности в браузерах:** Все современные браузеры помечают витрину и админку как «Не защищено» (Not Secure), что подрывает доверие покупателей и блокирует оформление заказов.
2. **Перехват конфиденциальных данных:** Пароли пользователей, данные профилей, токены сессий и учетные записи администраторов передаются по открытым каналам в открытом виде и уязвимы к атакам перехвата (Man-in-the-Middle).
3. **Блокировка современных веб-API:** Браузеры блокируют безопасные cookie-файлы (SameSite=None, Secure), Web Crypto API, аппаратную аутентификацию и PWA-функционал без HTTPS.
4. **Нестандартные порты в адресной строке:** Необходимость ввода `:3000` и `:3001` выглядит непрофессионально, ухудшает конверсию и часто блокируется корпоративными межсетевыми экранами.
5. **Невозможность получения сертификатов в текущей конфигурации:** Центры сертификации (Let's Encrypt) требуют для автоматической валидации (`http-01` challenge) обязательного ответа веб-сервера строго на 80 порту, который сейчас не слушается стеком.

---

## Solution

Развёртывание единого внешнего шлюза обратного проксирования (Edge Gateway) в составе инфраструктурного стека, который:
1. Слушает стандартные порты 80 (HTTP) и 443 (HTTPS) для внешних входящих запросов.
2. Автоматически перенаправляет весь обычный HTTP-трафик на защищённый HTTPS с кодом 301.
3. Прозрачно пропускает ACME-запросы валидации Let's Encrypt (`/.well-known/acme-challenge/`) на порт 80 без перенаправления.
4. Автоматически получает доверенные бесплатные TLS/SSL-сертификаты для витрины и админки через Certbot.
5. Выполняет безопасную терминацию SSL и маршрутизирует запросы на основе доменного имени (SNI / заголовок Host):
   - Запросы к основному домену витрины направляются во внутренний контейнер клиентской части.
   - Запросы к поддомену управления направляются во внутренний контейнер административной панели.
6. Полностью скрывает внутренние порты сервисов (3000, 3001, 8080, 5432, 6379) от внешнего публичного доступа, оставляя их доступными только внутри изолированной сети контейнеров.
7. Адаптирован под прерывистый график работы сервера (1–2 часа в день): выполняет проверку и продление сертификатов сразу при старте контейнеров (On-Boot) и каждый час работы, гарантируя своевременное продление без риска протухания.
8. Решает проблему «курицы и яйца» при первом развёртывании с помощью управляемой генерации временных сертификатов для успешного первого старта веб-сервера.

---

## User Stories

1. As a customer, I want to open the storefront using standard `https://` without port numbers, so that I have a clean and trustworthy shopping experience.
2. As a customer, I want my browser to display a valid green/neutral secure padlock without security warnings, so that I feel safe entering my personal data and placing orders.
3. As a customer, I want to automatically be redirected from `http://` to `https://` if I type the address without a protocol, so that my connection is always upgraded to secure.
4. As a customer, I want my authentication credentials and session tokens to be transmitted exclusively over an encrypted TLS connection, so that they cannot be intercepted in public Wi-Fi networks.
5. As an administrator, I want to access the administrative dashboard via a dedicated subdomain (e.g. `admin.domain.com`) over HTTPS on standard port 443, so that I do not need to memorize or enter port 3001.
6. As an administrator, I want administrative login credentials and elevated session tokens to be protected by TLS termination, so that store operations cannot be compromised via packet sniffing.
7. As a store manager, I want third-party payment gateways and webhook providers to successfully deliver status updates over valid HTTPS, so that order statuses are updated in real time.
8. As a DevOps engineer, I want Let's Encrypt certificates to be automatically acquired and renewed via Certbot without manual administrative intervention every 90 days.
9. As a DevOps engineer, I want the certificate renewal process to run immediately when the server boots up (On-Boot), so that the intermittent operating schedule of 1–2 hours per day does not cause certificates to expire.
10. As a DevOps engineer, I want certificate renewal checks to recur hourly while the machine remains online, so that any pending renewal is captured during extended operating sessions.
11. As a DevOps engineer, I want the web server to gracefully reload updated certificates in-memory without requiring elevated Docker daemon socket mounts inside the renewal container.
12. As a DevOps engineer, I want ports 3000, 3001, 8080, and database ports to be completely closed to public external access, so that unauthorized traffic cannot bypass HTTPS or access unhardened services.
13. As a DevOps engineer, I want a bootstrap procedure with temporary self-signed certificates, so that the web server can boot cleanly on a fresh server before Certbot has executed.
14. As a DevOps engineer, I want to configure storefront and admin domain names via environment variables, so that the stack can be deployed across testing, staging, and production environments without changing configuration files.
15. As a DevOps engineer, I want a staging toggle for Let's Encrypt, so that initial configuration tests can be executed against the ACME staging environment without consuming production rate limits.
16. As a frontend developer, I want the storefront SPA to continue communicating with the API via relative paths (`/api/v1`) without CORS errors, so that no changes to frontend source code or build configuration are required.
17. As an admin panel developer, I want file uploads and large media payloads to pass through the edge proxy without size restriction failures, so that catalog and media management works seamlessly.
18. As an infrastructure engineer, I want the entire gateway and certificate lifecycle configuration to reside in `vstore-infra` and validate through existing CI checks, so that deployment remains fully automated and version-controlled.

---

## Implementation Decisions

1. **Edge Reverse Proxy Architecture:**
   - A dedicated Edge Reverse Proxy service will be added to the infrastructure orchestration layer.
   - The proxy will be the sole service bound to host public ports 80 and 443 on all network interfaces.
   - Host bindings for ports 3000 and 3001 will be removed from the storefront and administrative container definitions, confining them strictly to the internal application bridge network.
   - The edge proxy will terminate incoming TLS connections and proxy requests internally to the existing storefront and administrative services over the internal network using standard HTTP/1.1 and preserving WebSocket/upgrade headers.

2. **Domain Routing & Virtual Hosts:**
   - Traffic routing will be determined by the HTTP `Host` header (Server Name Indication):
     - Requests addressed to the primary domain will be forwarded to the storefront service.
     - Requests addressed to the admin subdomain will be forwarded to the administrative dashboard service.
   - Inbound HTTP traffic on port 80 will be configured with a 301 Permanent Redirect to the corresponding HTTPS URI, with an explicit exception for ACME validation challenges.

3. **ACME Challenge Routing:**
   - The ACME `http-01` challenge mechanism will be used for domain validation.
   - A dedicated shared volume will be mounted between the edge proxy and the certificate management agent.
   - Requests matching the path `/.well-known/acme-challenge/` on port 80 will be served directly from the shared challenge directory and will not be redirected to HTTPS.

4. **Bootstrap Strategy ("Chicken-and-Egg" Resolution):**
   - An automated initialization script will be provided for fresh deployments.
   - If certificate files for configured domains do not exist in the certificates volume, the script will generate temporary self-signed (dummy) certificates to allow the web server to pass syntax checks and start successfully.
   - Once the web server is operational and listening on port 80, the certificate management agent will issue genuine Let's Encrypt certificates and trigger a graceful configuration reload.

5. **Intermittent Server Lifecycle Adaptation (1–2h Daily Operating Profile):**
   - The certificate management service will execute a certificate renewal check immediately upon container startup (On-Boot).
   - Following startup, the renewal agent will run an ongoing verification cycle at one-hour intervals while the host remains operational.
   - Because certificate renewal checks are lightweight and exit immediately when renewal is not due, this approach prevents certificate expiration despite non-continuous uptime.
   - The edge proxy will perform periodic background configuration reloads to pick up renewed certificates without exposing the Docker daemon socket to the renewal container.

6. **Environment-Driven Configuration & Rate Limit Safeguards:**
   - All domain names, administrator contact email, and ACME staging flags will be defined in the infrastructure environment template.
   - Template-based configuration substitution will generate the active proxy configuration at startup from environment variables.
   - A staging flag will be supported to route initial validation attempts to the Let's Encrypt staging environment to protect against rate limits.

---

## Testing Decisions

- **Definition of a Good Test:**
  Tests must verify the external behavioral contracts of the edge gateway (network responses, status codes, headers, and protocol behavior) from the perspective of an external client, without asserting internal container implementation details.

- **Modules to be Tested:**
  - Edge Reverse Proxy public interfaces (ports 80 and 443).
  - Certificate renewal agent command interface.
  - Infrastructure compose configuration validity.

- **Test Verifications (Seam: Edge Gateway HTTP/HTTPS Boundary):**
  1. *Compose Configuration Verification:*
     - Existing CI pipeline test `docker compose config --quiet` must succeed with the updated compose definition and environment template.
  2. *HTTP-to-HTTPS Redirection:*
     - A request to `http://<domain>/` on port 80 must return status code `301 Moved Permanently` with a `Location: https://<domain>/` header.
  3. *ACME Challenge Path Handling:*
     - A request to `http://<domain>/.well-known/acme-challenge/<test-token>` on port 80 must not redirect to HTTPS and must attempt to serve the challenge token from the shared webroot.
  4. *TLS Handshake & Host-based Routing:*
     - An HTTPS request with SNI for the storefront domain must return the storefront index response.
     - An HTTPS request with SNI for the admin domain must return the admin dashboard index response.
     - Proxied requests must carry `X-Forwarded-Proto: https`, `X-Forwarded-For`, and `Host` headers.
  5. *Port Isolation:*
     - Attempts to connect directly to ports 3000 and 3001 from outside the Docker network must be refused or unreachable.
  6. *Renewal Dry-Run:*
     - Executing the renewal command with the dry-run flag must report successful simulation without error.

- **Prior Art in Codebase:**
  - Existing compose validation in the infrastructure CI workflow (`ci.yml`).
  - Container health check probes currently defined in `docker-compose.yml`.

---

## Out of Scope

1. **DNS Record Management:** Configuring A-records with domain registrars or DNS providers is the responsibility of the domain owner.
2. **Dynamic DNS (DDNS) Automation:** The production host is confirmed to have a static public IP; automated IP tracking is out of scope.
3. **Internal Service-to-Service Encryption (mTLS):** Inter-container traffic between proxy, backend, database, and Redis within the private Docker bridge network remains unencrypted HTTP/TCP.
4. **Modifications to Application Source Repositories:** No changes to `vstore`, `vstore-front`, or `vstore-admin` codebases or Dockerfiles; all changes are contained within `vstore-infra`.
5. **Wildcard Certificates via DNS-01 Challenge:** Using DNS provider API tokens for DNS-01 validation is out of scope; validation uses HTTP-01 challenge.

---

## Further Notes

- **Triage Label:** `ready-for-agent`
- **Downstream Workflow:** This PRD can now be decomposed into atomic, independently actionable tasks using `/to-issues` or implemented directly via a feature branch (`feat/ssl-certbot-gateway`) in `vstore-infra` following the PR-only workflow.
