# Git: оформление веток и коммитов

## Ветки

Все изменения выполняются в отдельных ветках и попадают в `main` через Pull Request. Прямые коммиты в `main` запрещены.

Формат имени:

```text
<type>/<short-description>
<type>/<business-group>-<issue-number>-<short-description>
```

Тип обязателен; группа и номер задачи указываются при наличии. Описание — до 5–6 слов на английском, разделённых дефисами; оно должно обозначать конкретное изменение.

```text
feat/DEV-666-add-custom-input
fix/DEV-1125-issue-with-user-avatar-dimensions
chore/DEV-25-upgrade-vite-version
style/add-outline-to-primary-button
```

## Типы веток и коммитов

| Тип | Назначение |
| --- | --- |
| `feat` | Новая функциональность |
| `fix` | Исправление ошибки |
| `perf` | Улучшение производительности |
| `refactor` | Рефакторинг без изменения поведения |
| `style` | Стилизация и форматирование |
| `test` | Добавление или исправление тестов |
| `build` | Изменения сборки: npm, Vite и т. п. |
| `ci` | Изменения CI/CD |
| `docs` | Документация |
| `chore` | Зависимости, настройка проекта и инструментов |
| `revert` | Отмена коммита |

## Коммиты

Один коммит — одно законченное логическое изменение. Незавершённую работу сохраняйте как WIP.

```text
<type>: <short description>
```

Тип выбирается по содержимому коммита. После двоеточия — пробел. Описание — на английском, с глагола в повелительном наклонении (`add`, `remove`, `fix`, `update`), без знака препинания в конце.

```text
feat: add booking widget to product page
fix: correct user avatar dimensions
refactor: extract tariff calculation
```

При необходимости добавьте тело с пояснениями и подвал с метаданными, отделив их пустыми строками. Редактор сообщения открывается командой `git commit`.

```text
feat: add button for loading new posts

Hide the button when no more posts are available.

Issue: DEV-123
Signed-off-by: John Doe
```

### Трейлеры

Трейлеры — строки `Key: Value` в подвале сообщения: задача (`Issue`), этап (`Milestone`), подпись (`Signed-off-by`) и другие метаданные.

Алиасы ключей настраиваются через `git config`; трейлер добавляется опцией `--trailer`:

```bash
git config --global trailer.issue.key 'Issue'
git config --global trailer.sign.key 'Signed-off-by'
git config --global trailer.milestone.key 'Milestone'
git commit -m 'feat: add new counter' --trailer 'issue: DEV-111'
git commit --trailer 'sign: John Doe'
```

Поиск по сообщению и вывод выбранного трейлера:

```bash
git log --grep='Issue: DEV-111'
git log --format='%h %(trailers:key=Issue)' --grep='DEV-111'
```

## WIP-коммиты

WIP (Work In Progress) — промежуточный коммит для сохранения незавершённой работы.

- В конце рабочего дня сохраните незавершённые изменения в WIP-коммит и отправьте в свою удалённую ветку.
- Перед слиянием PR объедините WIP-коммиты в законченные логические изменения и оформите обычные сообщения. WIP не должен попадать в `main`.

```bash
git commit -m 'WIP: implement product card'
git push
```

Для объединения используйте интерактивный rebase относительно базовой ветки:

```bash
git rebase -i <base-branch>
```

Коммиты в списке идут от старых к новым. Чтобы объединить три коммита, переименуйте первый через `reword`, остальные присоедините через `fixup`:

```text
reword f7f3f6d WIP: implement product card
fixup 310154e WIP: update card styles
fixup a5f4a0d WIP: update card skeleton
```

| Команда | Действие |
| --- | --- |
| `pick` | Сохранить коммит |
| `reword` | Изменить сообщение |
| `squash` | Объединить с предыдущим, сохранив сообщения для редактирования |
| `fixup` | Объединить с предыдущим, отбросив сообщение текущего |
| `drop` | Удалить коммит |

Сохраните список и задайте итоговое сообщение, например `feat: add product card`. Если ветка уже опубликована, обновите её после rebase:

```bash
git push --force-with-lease
```

Переписывайте историю только своей рабочей ветки. `--force-with-lease` отклонит отправку, если удалённая ветка отличается от ожидаемого состояния.

Подробнее: [Conventional Commits](https://www.conventionalcommits.org/ru/v1.0.0/), [Git trailers](https://git-scm.com/docs/git-interpret-trailers), [переписывание истории](https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History). Для проверки сообщений можно использовать [Commitlint](https://commitlint.js.org/#/).
