# VStore Docs

Общие правила, инженерные планы и документация VStore.

- [План развития инженерных процессов](PM/TeamLead-plan.md).
- [Распределение ответственности за выполнение плана](PM/TeamLead-responsibility.md).
- [Владельцы компонентов и выбор reviewer](CONTRIBUTING.md) — TL-01, назначения частично ожидают подтверждения.
- [Единый процесс веток, review и merge](CONTRIBUTING.md#единый-процесс-изменений--tl-02) — TL-02, предложен к утверждению.
- [PR template для UI, CLI и AI](CONTRIBUTING.md#pr-template-для-github-ui-cli-и-ai--tl-03).
- [Issue tracker](agents/issue-tracker.md), [triage labels](agents/triage-labels.md), [domain docs](agents/domain.md).
- [Private deployment permissions, secrets and snapshots](infra/deployment-permissions.md) — A16.
- [Edge image security gate, adoption and rollback](infra/edge-security.md) — A05.
- [Upstream recovery and HTTPS API deployment gates](infra/upstream-recovery.md) — A02.
- [Trusted client identity: proxy trust, rollout and backend handoff](infra/client-identity.md) — A03.
- [PRD: SSL/Certbot gateway](prd/001-ssl-certbot-gateway.md) и [декомпозиция](prd/001-ssl-certbot-gateway-slices.md).

## Репозитории компонентов

- [Backend](https://github.com/vstore-ex/vstore).
- [Storefront](https://github.com/vstore-ex/vstore-front).
- [Admin](https://github.com/vstore-ex/vstore-admin).
- [Infrastructure](https://github.com/vstore-ex/vstore-infra).

## Изменения документации

Изменения выполняются в feat/*, fix/* или chore/* и передаются через PR.
Автор проверяет ссылки и соответствие существующей конфигурации. Изменение
ответственности подтверждается участниками, которых оно касается.
GitHub Issues хранит актуальный статус задач; план задаёт требования и порядок работы.
Исторические audit-artifacts в планах относятся к локальному workspace и не включены
в этот репозиторий. Актуальные результаты проверяются по связанным Issues/PR.
