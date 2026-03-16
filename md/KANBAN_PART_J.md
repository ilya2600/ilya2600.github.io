{% raw %}
# Часть J: Журнал активности и архивирование / удаление досок

В Частях F–I вы реализовали:

- **доски** (`boards`) с владельцем, правами доступа и списком на дашборде;
- **колонки** (`columns`) с WIP‑лимитами;
- **карточки** (`cards`) с добавлением, редактированием, перемещением и базовым применением WIP;
- **участников доски** (`collaborators`) с ролями `owner` / `editor` / `viewer` и раздел «Доски, в которых вы участвуете».

В этой завершающей части вы:

- добавите **журнал активности карточек**;
- реализуете **архивирование**, **восстановление** и **удаление** доски;
- сделаете так, чтобы **архивная доска была доступна только для чтения**.

---

## Цель Части J

После выполнения этой части у вас будет:

- таблица `card_activity` в базе данных и хелпер `_log_activity`, вызываемый при создании, редактировании и перемещении карточек;
- раздел «Недавняя активность» на странице доски;
- кнопки:
  - «Архивировать» / «Разархивировать» доску (только владелец),
  - «Удалить доску» (только владелец, с подтверждением);
- обновлённый дашборд с разделом «Архивные доски»;
- поведение «только чтение» для архивных досок:
  - нельзя добавлять колонки и карточки,
  - нельзя редактировать или перемещать карточки,
  - участники видят содержимое и историю, но не могут его менять.

**Видимое / проверяемое поведение:**

- при создании, редактировании и перемещении карточек на странице доски появляются записи в блоке «Недавняя активность»;
- владелец может архивировать доску:
  - она исчезает из основных списков и появляется в разделе «Архивные доски»,
  - доска в архиве становится доступной только для чтения;
- владелец может восстановить доску из архива;
- владелец может полностью удалить доску (вместе с колонками, карточками, участниками и активностью).

---

## J1. Таблица `card_activity` в базе данных

Сначала создадим таблицу, которая будет хранить историю действий с карточками.

### Шаг J1.1. Добавляем `card_activity` в `init_db`

Откройте `db.py` и найдите функцию:

```python
def init_db():
    ...
```

Внутри неё уже создаются:

- `settings`,
- `users`,
- `boards`,
- `columns`,
- `cards`,
- `collaborators`.

Сразу **после** блока создания `collaborators` добавьте:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS card_activity (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            card_id INTEGER NOT NULL,
            user_id INTEGER NOT NULL,
            action_type TEXT NOT NULL CHECK (action_type IN ('created', 'edited', 'moved')),
            from_column_id INTEGER,
            to_column_id INTEGER,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            FOREIGN KEY (card_id) REFERENCES cards(id),
            FOREIGN KEY (user_id) REFERENCES users(id),
            FOREIGN KEY (from_column_id) REFERENCES columns(id),
            FOREIGN KEY (to_column_id) REFERENCES columns(id)
        )
    """)
```

Пояснения:

- `card_id` — к какой карточке относится событие;
- `user_id` — кто выполнял действие;
- `action_type` — строка `'created'`, `'edited'` или `'moved'`;
- `from_column_id` / `to_column_id` — для перемещения карточки:
  - при создании / редактировании могут быть `NULL`;
- `created_at` — когда произошло действие.

> Как и прежде, мы не включаем каскадное удаление напрямую в схему. Дальше вы реализуете ручное удаление связанных данных при удалении доски.

---

## J2. Хелпер `_log_activity` и использование в `views_boards.py`

Теперь реализуем функцию, которую будем вызывать в местах изменения карточек.

### Шаг J2.1. Реализуем `_log_activity`

Откройте `views_boards.py` и рядом с уже существующими хелперами для карточек (например, рядом с `count_cards_in_column`) добавьте:

```python
def _log_activity(conn, card_id: int, user_id: int, action_type: str, from_column_id=None, to_column_id=None):
    """Залогировать действие с карточкой в таблице card_activity."""
    if action_type not in ("created", "edited", "moved"):
        return

    conn.execute(
        """
        INSERT INTO card_activity (card_id, user_id, action_type, from_column_id, to_column_id)
        VALUES (?, ?, ?, ?, ?)
        """,
        (card_id, user_id, action_type, from_column_id, to_column_id),
    )
```

Эта функция:

- не открывает и не закрывает соединение — использует уже существующее `conn`;
- проверяет `action_type`, чтобы не записать что‑то случайное;
- записывает одну строку в `card_activity`.

### Шаг J2.2. Вызываем `_log_activity` при создании карточки

В `card_create_view` после вставки новой карточки у нас уже есть `card_id` **не напрямую** — сейчас код вставляет карточку без возврата ID. Обновим его так, чтобы сразу получить `id` созданной карточки и записать лог.

Найдите `card_create_view` и замените фрагмент:

```python
    conn.execute(
        """
        INSERT INTO cards (column_id, title, description, position, created_by)
        VALUES (?, ?, ?, ?, ?)
        """,
        (column_id, title, description or "", next_position, user["id"]),
    )
    conn.commit()
    conn.close()

    return redirect(url_for("board_view", board_id=board_id))
```

на:

```python
    cur = conn.execute(
        """
        INSERT INTO cards (column_id, title, description, position, created_by)
        VALUES (?, ?, ?, ?, ?)
        """,
        (column_id, title, description or "", next_position, user["id"]),
    )
    card_id = cur.lastrowid

    _log_activity(conn, card_id=card_id, user_id=user["id"], action_type="created", from_column_id=None, to_column_id=column_id)

    conn.commit()
    conn.close()

    return redirect(url_for("board_view", board_id=board_id))
```

### Шаг J2.3. Логируем редактирование карточки

В `card_edit_view` после успешного `UPDATE` добавим запись в журнал.

Найдите фрагмент:

```python
    conn.execute(
        """
        UPDATE cards
        SET title = ?, description = ?, updated_at = (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        WHERE id = ?
        """,
        (title, description or "", card_id),
    )
    conn.commit()
    conn.close()

    return redirect(url_for("board_view", board_id=board_id))
```

и замените на:

```python
    conn.execute(
        """
        UPDATE cards
        SET title = ?, description = ?, updated_at = (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        WHERE id = ?
        """,
        (title, description or "", card_id),
    )

    user = current_user()
    if user is not None:
        _log_activity(conn, card_id=card_id, user_id=user["id"], action_type="edited", from_column_id=None, to_column_id=None)

    conn.commit()
    conn.close()

    return redirect(url_for("board_view", board_id=board_id))
```

### Шаг J2.4. Логируем перемещение карточки

В `card_move_view` нам важно знать:

- из какой колонки (`current_column_id`) в какую (`new_column_id`) переносится карточка.

Найдите конец функции `card_move_view` (часть, где выполняется `UPDATE`) и модифицируйте его.

Было:

```python
    conn.execute(
        """
        UPDATE cards
        SET column_id = ?, position = ?, updated_at = (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        WHERE id = ?
        """,
        (new_column_id, next_position, card_id),
    )
    conn.commit()
    conn.close()

    return redirect(url_for("board_view", board_id=board_id))
```

Сделайте так:

```python
    conn.execute(
        """
        UPDATE cards
        SET column_id = ?, position = ?, updated_at = (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        WHERE id = ?
        """,
        (new_column_id, next_position, card_id),
    )

    user = current_user()
    if user is not None:
        _log_activity(
            conn,
            card_id=card_id,
            user_id=user["id"],
            action_type="moved",
            from_column_id=current_column_id,
            to_column_id=new_column_id,
        )

    conn.commit()
    conn.close()

    return redirect(url_for("board_view", board_id=board_id))
```

---

## J3. Вывод журнала активности на странице доски

Теперь выведем «Недавнюю активность» внизу страницы доски.

### Шаг J3.1. Хелпер `recent_activity_for_board`

В `views_boards.py` добавьте функцию, которая по `board_id` вернёт последние события:

```python
def recent_activity_for_board(board_id: int, limit: int = 30):
    """Вернуть последние события по карточкам этой доски."""
    conn = get_conn()
    rows = conn.execute(
        """
        SELECT
            a.id,
            a.card_id,
            a.user_id,
            a.action_type,
            a.from_column_id,
            a.to_column_id,
            a.created_at,
            u.username AS user_username,
            c.title AS card_title,
            col_from.name AS from_column_name,
            col_to.name AS to_column_name
        FROM card_activity a
        JOIN cards c ON a.card_id = c.id
        JOIN users u ON a.user_id = u.id
        JOIN columns col_card ON c.column_id = col_card.id
        JOIN boards b ON col_card.board_id = b.id
        LEFT JOIN columns col_from ON a.from_column_id = col_from.id
        LEFT JOIN columns col_to ON a.to_column_id = col_to.id
        WHERE b.id = ?
        ORDER BY a.created_at DESC, a.id DESC
        LIMIT ?
        """,
        (board_id, limit),
    ).fetchall()
    conn.close()

    return rows
```

### Шаг J3.2. Передаём активность в шаблон `board.html`

В `board_view_view` после загрузки `columns`, `cards_by_column` и `collaborators` добавьте:

```python
    activity = recent_activity_for_board(board_id)
```

и передайте `activity=activity` в `render_template`:

```python
    return render_template(
        "board.html",
        board=board,
        columns=columns,
        cards_by_column=cards_by_column,
        collaborators=collaborators,
        available_users=available_users,
        activity=activity,
        is_owner=is_board_owner(board_id),
        can_edit=can_edit,
    )
```

### Шаг J3.3. Раздел «Недавняя активность» в `board.html`

Откройте `templates/board.html`. Внизу основного `<section class="card">...</section>` (например, перед ссылкой «К дашборду») добавьте:

```html
    <h2>Недавняя активность</h2>

    {% if not activity %}
      <p class="muted">Пока нет зафиксированных действий.</p>
    {% else %}
      <ul class="activity-list">
        {% for a in activity %}
          <li class="activity-item">
            <span class="activity-time">{{ a.created_at }}</span>
            —
            <strong>{{ a.user_username }}</strong>
            {% if a.action_type == 'created' %}
              создал(а) карточку
            {% elif a.action_type == 'edited' %}
              изменил(а) карточку
            {% elif a.action_type == 'moved' %}
              переместил(а) карточку
            {% else %}
              выполнил(а) действие
            {% endif %}
            «{{ a.card_title }}»
            {% if a.action_type == 'moved' %}
              {% if a.from_column_name and a.to_column_name %}
                из «{{ a.from_column_name }}» в «{{ a.to_column_name }}»
              {% elif a.to_column_name %}
                в «{{ a.to_column_name }}»
              {% endif %}
            {% endif %}
          </li>
        {% endfor %}
      </ul>
    {% endif %}
```

Теперь каждый раз, когда вы создаёте, редактируете или перемещаете карточку, в этом списке появляется новая строка.

---

## J4. Архивирование и восстановление доски

Следующая задача — позволить владельцу помечать доску как «в архиве» и возвращать её обратно.

### Шаг J4.1. View‑функции архивации и восстановления

В `views_boards.py` добавьте две функции:

```python
def board_archive_view(board_id: int):
    """Архивировать доску (только владелец)."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    if not is_board_owner(board_id):
        abort(404)

    conn = get_conn()
    conn.execute(
        "UPDATE boards SET is_archived = 1 WHERE id = ?",
        (board_id,),
    )
    conn.commit()
    conn.close()

    return redirect(url_for("dashboard"))


def board_restore_view(board_id: int):
    """Разархивировать доску (только владелец)."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    if not is_board_owner(board_id):
        abort(404)

    conn = get_conn()
    conn.execute(
        "UPDATE boards SET is_archived = 0 WHERE id = ?",
        (board_id,),
    )
    conn.commit()
    conn.close()

    return redirect(url_for("board_view", board_id=board_id))
```

### Шаг J4.2. Маршруты для архивации и восстановления

В `app.py` импортируйте эти вьюхи:

```python
from views_boards import (
    board_new_view,
    board_create_view,
    board_view_view,
    column_create_view,
    card_create_view,
    card_edit_view,
    card_move_view,
    collaborator_add_view,
    collaborator_remove_view,
    board_archive_view,
    board_restore_view,
)
```

И добавьте маршруты:

```python
@app.post("/boards/<int:board_id>/archive")
def board_archive(board_id):
    return board_archive_view(board_id)


@app.post("/boards/<int:board_id>/restore")
def board_restore(board_id):
    return board_restore_view(board_id)
```

### Шаг J4.3. Кнопки на странице доски

В `board.html`, в секции для владельца (`{% if is_owner %}`), добавьте кнопки архивации / восстановления. Например, над разделом «Участники»:

```html
    {% if is_owner %}
      <section class="board-actions">
        <h2>Управление доской</h2>

        <form method="post" action="{{ url_for('board_archive', board_id=board.id) }}" style="display: inline;">
          {% if not board.is_archived %}
            <button type="submit" class="btn btn-secondary">Архивировать доску</button>
          {% endif %}
        </form>

        <form method="post" action="{{ url_for('board_restore', board_id=board.id) }}" style="display: inline;">
          {% if board.is_archived %}
            <button type="submit" class="btn btn-secondary">Разархивировать доску</button>
          {% endif %}
        </form>
      </section>
    {% endif %}
```

> Обратите внимание: логика «только чтение» для архивных досок вы реализуете в следующем шаге, отключив формы редактирования при `board.is_archived`.

---

## J5. Удаление доски и связанных данных

Теперь реализуем полное удаление доски. Поскольку мы не включили каскадное удаление в схеме, нам нужно вручную удалить:

- `card_activity` карточек этой доски,
- `cards`,
- `columns`,
- `collaborators`,
- саму запись в `boards`.

### Шаг J5.1. View‑функция удаления `board_delete_view`

В `views_boards.py` добавьте:

```python
def board_delete_view(board_id: int):
    """Полностью удалить доску и все связанные данные (только владелец)."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    if not is_board_owner(board_id):
        abort(404)

    conn = get_conn()

    # Находим все колонки и карточки этой доски
    columns = conn.execute(
        "SELECT id FROM columns WHERE board_id = ?",
        (board_id,),
    ).fetchall()
    column_ids = [col["id"] for col in columns]

    card_ids = []
    if column_ids:
        rows = conn.execute(
            "SELECT id FROM cards WHERE column_id IN ({})".format(
                ",".join("?" for _ in column_ids)
            ),
            column_ids,
        ).fetchall()
        card_ids = [r["id"] for r in rows]

    # Удаляем активность по карточкам
    if card_ids:
        conn.execute(
            "DELETE FROM card_activity WHERE card_id IN ({})".format(
                ",".join("?" for _ in card_ids)
            ),
            card_ids,
        )

    # Удаляем карточки и колонки
    if column_ids:
        conn.execute(
            "DELETE FROM cards WHERE column_id IN ({})".format(
                ",".join("?" for _ in column_ids)
            ),
            column_ids,
        )
        conn.execute(
            "DELETE FROM columns WHERE board_id = ?",
            (board_id,),
        )

    # Удаляем участников
    conn.execute(
        "DELETE FROM collaborators WHERE board_id = ?",
        (board_id,),
    )

    # Удаляем саму доску
    conn.execute(
        "DELETE FROM boards WHERE id = ?",
        (board_id,),
    )

    conn.commit()
    conn.close()

    return redirect(url_for("dashboard"))
```

> Замечание: здесь используется динамическая подстановка плейсхолдеров `?` для списка `IN (...)`. Важно формировать строку плейсхолдеров отдельно, а значения передавать вторым аргументом в `execute`.

### Шаг J5.2. Маршрут и кнопка удаления

В `app.py` импортируйте `board_delete_view`:

```python
from views_boards import (
    ...,
    board_delete_view,
)
```

и добавьте маршрут:

```python
@app.post("/boards/<int:board_id>/delete")
def board_delete(board_id):
    return board_delete_view(board_id)
```

В `board.html`, в секции «Управление доской» для владельца, добавьте кнопку удаления (лучше с простым подтверждением через `onsubmit`):

```html
        <form method="post" action="{{ url_for('board_delete', board_id=board.id) }}" style="display: inline;"
              onsubmit="return confirm('Вы уверены, что хотите удалить эту доску без возможности восстановления?');">
          <button type="submit" class="btn btn-secondary">Удалить доску</button>
        </form>
```

---

## J6. Архивные доски: раздел на дашборде и режим «только чтение»

Осталось:

- показать архивные доски на дашборде;
- убедиться, что при `board.is_archived` доска только для чтения.

### Шаг J6.1. Архивные доски на дашборде

В `views_boards.py` можно добавить хелпер:

```python
def archived_boards_for_user():
    """Вернуть доски пользователя, которые находятся в архиве (где он владелец или участник)."""
    user = current_user()
    if user is None:
        return []

    conn = get_conn()
    rows = conn.execute(
        """
        SELECT DISTINCT
            b.id,
            b.title,
            b.description,
            b.is_archived,
            b.created_at
        FROM boards b
        LEFT JOIN collaborators c
            ON c.board_id = b.id
        WHERE
            b.is_archived = 1
            AND (
                b.owner_id = ?
                OR c.user_id = ?
            )
        ORDER BY b.created_at DESC
        """,
        (user["id"], user["id"]),
    ).fetchall()
    conn.close()

    return rows
```

В `views_auth.dashboard_view` импортируйте и используйте его:

```python
from views_boards import boards_for_user, collaborator_boards_for_user, archived_boards_for_user
```

и дополните `dashboard_view`:

```python
    boards = boards_for_user()
    collaborator_boards = collaborator_boards_for_user()
    archived_boards = archived_boards_for_user()

    return render_template(
        "dashboard.html",
        user=user,
        boards=boards,
        collaborator_boards=collaborator_boards,
        archived_boards=archived_boards,
    )
```

В `templates/dashboard.html` добавьте новый раздел:

```html
    <h2 style="margin-top: 32px;">Архивные доски</h2>

    {% if not archived_boards %}
      <p class="muted">У вас пока нет архивных досок.</p>
    {% else %}
      <table class="table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Название</th>
            <th>Описание</th>
            <th>Создана</th>
          </tr>
        </thead>
        <tbody>
          {% for b in archived_boards %}
            <tr>
              <td><a href="{{ url_for('board_view', board_id=b.id) }}">{{ b.id }}</a></td>
              <td><a href="{{ url_for('board_view', board_id=b.id) }}">{{ b.title }}</a></td>
              <td>{{ b.description }}</td>
              <td>{{ b.created_at[:10] if b.created_at else '' }}</td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    {% endif %}
```

### Шаг J6.2. Только чтение для архивных досок

Мы уже выводим бейдж «В архиве» в `board.html`, опираясь на `board.is_archived`. Теперь нужно скрыть все формы редактирования, если доска в архиве.

В `board.html`:

- все условия вида `{% if can_edit %}` замените на `{% if can_edit and not board.is_archived %}`;
- секции «Добавить карточку», «Редактировать», «Перенести в колонку», форма «Добавить колонку» и форма «Добавить участника» (по желанию) должны быть видны только если доска **не** в архиве.

Пример:

```html
    {% if can_edit and not board.is_archived %}
      <section class="add-column">
        ...
      </section>
    {% endif %}
```

Для владельца архивной доски:

- кнопка «Разархивировать доску» остаётся доступной;
- кнопка «Удалить доску» тоже может быть доступна;
- кнопка «Архивировать доску» должна быть скрыта, если `board.is_archived` уже `1`.

---

## J7. Проверка активности и архива

После всех изменений перезапустите приложение и проверьте:

1. **Журнал активности.**
   - Создайте доску, несколько колонок и карточек.
   - На странице доски внизу должен появиться блок «Недавняя активность».
   - При создании карточки — новая строка «создал(а) карточку …».
   - При редактировании — «изменил(а) карточку …».
   - При перемещении — «переместил(а) карточку … из X в Y».
2. **Архивирование.**
   - Под владельцем нажмите «Архивировать доску».
   - Доска должна исчезнуть из разделов «Ваши доски» и «Доски, в которых вы участвуете» и появиться в «Архивных досках».
   - При открытии архивной доски:
     - должен быть бейдж «В архиве»,
     - не должно быть форм добавления / редактирования / перемещения (только просмотр),
     - раздел активности остаётся видимым.
3. **Восстановление.**
   - Нажмите «Разархивировать доску».
   - Доска должна вернуться в список «Ваши доски» / «Доски, в которых вы участвуете» и снова стать редактируемой.
4. **Удаление.**
   - Нажмите «Удалить доску» (под владельцем) и подтвердите.
   - Доска должна пропасть из всех разделов дашборда.
   - Попытка открыть её по прямой ссылке должна приводить к 404.

Если всё работает, как описано, — Часть J завершена, а поведение приложения соответствует целям `KANBAN_FINAL.md`.

---

## Итог Части J (Активность и архив)

После выполнения этой части у вас есть:

- таблица `card_activity` и хелпер `_log_activity`, вызываемый при создании, редактировании и перемещении карточек;
- раздел «Недавняя активность» на доске с человекочитаемым описанием действий;
- возможности:
  - архивировать доску и возвращать её из архива (только владелец),
  - удалять доску вместе с колонками, карточками, участниками и журналом активности;
- обновлённый дашборд:
  - «Ваши доски» (активные, где вы владелец),
  - «Доски, в которых вы участвуете» (активные, где вы участник),
  - «Архивные доски» (где вы владелец или участник);
- логика «только чтение» для архивных досок: никакого редактирования, только просмотр содержимого и журнала.

На этом учебный Kanban‑проект достигает целевого функционала из `KANBAN_FINAL.md`.
{% endraw %}

