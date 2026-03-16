{% raw %}
# Часть H: Карточки — добавление, редактирование, перемещение и WIP

В Части G вы добавили на доску **колонки** (`columns`):

- таблица `columns` в базе данных;
- хелпер `columns_for_board(board_id)` в `views_boards.py`;
- обновлённый `board_view_view`, который передаёт в шаблон `board`, `columns`, `can_edit`, `is_owner`;
- маршруты и форма для добавления колонок;
- шаблон `board.html`, который показывает колонки и для колонок с WIP‑лимитом рисует «WIP: 0 / N» (пока без реального подсчёта карточек).

В этой части вы добавите **карточки** внутри колонок. После выполнения Части H:

- у каждой колонки появится список карточек;
- можно будет добавлять, редактировать и перемещать карточки между колонками одной доски;
- WIP‑лимиты колонок начнут реально работать: при попытке добавить / перенести карточку в переполненную колонку операция будет блокироваться.

---

## Цель Части H

После выполнения этой части у вас будет:

- таблица `cards` в базе данных, связанная с `columns` и пользователями;
- возможность добавлять карточки в колонку (с заголовком и описанием);
- возможность редактировать карточку;
- возможность перемещать карточку в другую колонку той же доски;
- базовая проверка WIP‑лимита:
  - при добавлении карточки в колонку,
  - при перемещении карточки в новую колонку.

**Видимое / проверяемое поведение:**

- на странице доски внутри каждой колонки виден список карточек;
- вы можете:
  - создать карточку в конкретной колонке → она появляется внизу списка этой колонки;
  - отредактировать карточку → заголовок и описание обновляются;
  - перенести карточку в другую колонку → она исчезает из старой и появляется в новой;
- при достижении WIP‑лимита:
  - новая карточка в колонку не создаётся;
  - карточка не может быть перенесена в колонку, у которой уже максимальное число карточек;
  - вместо этого вы видите сообщение о превышении лимита.

---

## H1. Таблица `cards` в инициализации базы

Сначала нужно добавить в базу данных таблицу `cards`. Она будет хранить:

- к какой колонке относится карточка;
- её заголовок и описание;
- позицию внутри колонки;
- кто и когда создал / изменил карточку.

### Шаг H1.1. Открываем `init_db` и находим место после `columns`

Откройте файл `db.py` и найдите функцию:

```python
def init_db():
    ...
```

Внутри неё уже должны создаваться таблицы `settings`, `users`, `boards` и `columns`. Нас интересует место сразу после блока:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS columns (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            board_id INTEGER NOT NULL,
            name TEXT NOT NULL,
            position INTEGER NOT NULL,
            wip_limit INTEGER,
            FOREIGN KEY (board_id) REFERENCES boards(id)
        )
    """)
```

### Шаг H1.2. Добавляем `CREATE TABLE IF NOT EXISTS cards`

Сразу **после** создания таблицы `columns` добавьте блок:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS cards (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            column_id INTEGER NOT NULL,
            title TEXT NOT NULL,
            description TEXT NOT NULL,
            position INTEGER NOT NULL,
            created_by INTEGER NOT NULL,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            updated_at TEXT,
            FOREIGN KEY (column_id) REFERENCES columns(id),
            FOREIGN KEY (created_by) REFERENCES users(id)
        )
    """)
```

Пояснения:

- `column_id` — колонка, к которой относится карточка;
- `title` — заголовок карточки (короткий текст);
- `description` — описание (может быть пустой строкой, но поле не `NULL` — для простоты);
- `position` — целое число для сортировки карточек сверху вниз (`ORDER BY position`);
- `created_by` — `id` пользователя, создавшего карточку;
- `created_at` — время создания (устанавливается автоматически);
- `updated_at` — время последнего обновления (устанавливается при редактировании / перемещении).

> Как и в Части G, пока мы **не включаем каскадное удаление** и не трогаем `PRAGMA foreign_keys`. К этим вопросам вы вернётесь в Части J, когда будете реализовывать окончательную логику удаления доски и связанных данных.

---

## H2. Подготовка `views_boards.py` к работе с карточками

Теперь нужно доработать `views_boards.py`, чтобы:

- загружать карточки для колонок доски;
- уметь добавлять, редактировать и перемещать карточки;
- проверять WIP‑лимиты при добавлении / перемещении.

### Шаг H2.1. Структура данных для карточек в `board_view_view`

Сейчас `board_view_view` загружает доску и список колонок:

```python
    columns = columns_for_board(board_id)
    can_edit = can_edit_board(board_id)

    return render_template(
        "board.html",
        board=board,
        columns=columns,
        is_owner=is_board_owner(board_id),
        can_edit=can_edit,
    )
```

Чтобы отрисовать карточки, есть два варианта:

1. загружать карточки «внутри шаблона» (неудобно и сложно тестировать);
2. подготовить структуру данных в Python и передать её в шаблон.

Мы выберем второй вариант.

Добавим в `views_boards.py` рядом с `columns_for_board` новый хелпер для карточек:

```python
def cards_for_board(board_id: int):
    """Вернуть все карточки доски, сгруппированные по колонкам."""
    conn = get_conn()
    rows = conn.execute(
        """
        SELECT
            c.id,
            c.column_id,
            c.title,
            c.description,
            c.position,
            c.created_by,
            c.created_at,
            c.updated_at
        FROM cards c
        JOIN columns col ON c.column_id = col.id
        WHERE col.board_id = ?
        ORDER BY c.position ASC, c.id ASC
        """,
        (board_id,),
    ).fetchall()
    conn.close()

    by_column = {}
    for row in rows:
        by_column.setdefault(row["column_id"], []).append(row)

    return by_column
```

Эта функция:

- выбирает все карточки всех колонок указанной доски;
- сортирует их по `position` (и по `id` на случай одинаковой позиции);
- группирует карточки по `column_id` в словарь `column_id → [cards...]`.

Теперь изменим `board_view_view`, чтобы он использовал новый хелпер:

```python
    columns = columns_for_board(board_id)
    cards_by_column = cards_for_board(board_id)
    can_edit = can_edit_board(board_id)

    return render_template(
        "board.html",
        board=board,
        columns=columns,
        cards_by_column=cards_by_column,
        is_owner=is_board_owner(board_id),
        can_edit=can_edit,
    )
```

> Важно: если вы уже изменили `board_view_view` в Части G, просто дополните его:
> добавьте вызов `cards_for_board` и параметр `cards_by_column=cards_by_column` в `render_template`.

### Шаг H2.2. Вспомогательная функция `count_cards_in_column`

Чтобы реализовать WIP‑лимиты, нам понадобится быстро узнавать количество карточек в колонке. Добавим простой хелпер:

```python
def count_cards_in_column(conn, column_id: int) -> int:
    """Подсчитать количество карточек в колонке (использует существующее соединение)."""
    row = conn.execute(
        "SELECT COUNT(*) AS cnt FROM cards WHERE column_id = ?",
        (column_id,),
    ).fetchone()
    return row["cnt"] if row is not None else 0
```

Эта функция:

- принимает уже открытое соединение `conn` (чтобы не открывать новое в каждом запросе);
- возвращает целое количество карточек в колонке.

Мы будем использовать её в обработчиках добавления и перемещения карточки.

---

## H3. Добавление карточки в колонку

Теперь реализуем создание карточек. Логика:

- форма создания карточки будет располагаться внутри каждой колонки;
- она отправляет POST‑запрос на маршрут вида `"/columns/<int:column_id>/cards/new"`;
- обработчик:
  - проверяет права (`can_edit_board` для доски этой колонки),
  - проверяет WIP‑лимит,
  - создаёт карточку и перенаправляет обратно на страницу доски.

### Шаг H3.1. View‑функция `card_create_view`

В `views_boards.py`, ниже `column_create_view`, добавьте:

```python
def card_create_view(column_id: int):
    """Создать новую карточку в колонке."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    conn = get_conn()

    # Находим колонку и связанную доску
    column = conn.execute(
        "SELECT id, board_id, wip_limit FROM columns WHERE id = ?",
        (column_id,),
    ).fetchone()

    if column is None:
        conn.close()
        abort(404)

    board_id = column["board_id"]

    if not can_edit_board(board_id):
        conn.close()
        abort(404)

    title = (request.form.get("title") or "").strip()
    description = (request.form.get("description") or "").strip()

    if not title:
        conn.close()
        return redirect(url_for("board_view", board_id=board_id))

    # Проверяем WIP‑лимит: если задан и колонка уже заполнена, не создаём карточку.
    if column["wip_limit"] is not None and column["wip_limit"] > 0:
        current_count = count_cards_in_column(conn, column_id)
        if current_count >= column["wip_limit"]:
            conn.close()
            # Позже можно заменить на flash; пока достаточно вернуть пользователя на доску.
            return redirect(url_for("board_view", board_id=board_id))

    user = current_user()

    # Находим следующий position в этой колонке
    row = conn.execute(
        "SELECT COALESCE(MAX(position), 0) AS max_pos FROM cards WHERE column_id = ?",
        (column_id,),
    ).fetchone()
    next_position = (row["max_pos"] or 0) + 1

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

### Шаг H3.2. Маршрут для создания карточки в `app.py`

Откройте `app.py` и дополните импорт из `views_boards`:

```python
from views_boards import (
    board_new_view,
    board_create_view,
    board_view_view,
    column_create_view,
    card_create_view,
)
```

Затем ниже маршрута `column_create` добавьте новый маршрут:

```python
@app.post("/columns/<int:column_id>/cards/new")
def card_create(column_id):
    return card_create_view(column_id)
```

---

## H4. Редактирование карточки

Теперь реализуем возможность менять заголовок и описание карточки.

Для простоты в этой части мы будем использовать:

- небольшую форму редактирования прямо под карточкой;
- отдельный POST‑маршрут `"/cards/<int:card_id>/edit"`.

### Шаг H4.1. View‑функция `card_edit_view`

В `views_boards.py`, под `card_create_view`, добавьте:

```python
def card_edit_view(card_id: int):
    """Изменить заголовок и описание карточки."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    conn = get_conn()

    card = conn.execute(
        """
        SELECT c.id, c.column_id, c.title, c.description, col.board_id
        FROM cards c
        JOIN columns col ON c.column_id = col.id
        WHERE c.id = ?
        """,
        (card_id,),
    ).fetchone()

    if card is None:
        conn.close()
        abort(404)

    board_id = card["board_id"]

    if not can_edit_board(board_id):
        conn.close()
        abort(404)

    title = (request.form.get("title") or "").strip()
    description = (request.form.get("description") or "").strip()

    if not title:
        conn.close()
        return redirect(url_for("board_view", board_id=board_id))

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

### Шаг H4.2. Маршрут редактирования карточки в `app.py`

В `app.py` расширьте импорт:

```python
from views_boards import (
    board_new_view,
    board_create_view,
    board_view_view,
    column_create_view,
    card_create_view,
    card_edit_view,
)
```

И ниже маршрута `card_create` добавьте:

```python
@app.post("/cards/<int:card_id>/edit")
def card_edit(card_id):
    return card_edit_view(card_id)
```

---

## H5. Перемещение карточки между колонками

Последний шаг — дать возможность перемещать карточки между колонками одной и той же доски с учётом WIP‑лимитов.

Интерфейс:

- под каждой карточкой будет форма с селектом «Переместить в колонку» и кнопкой «Перенести»;
- форма отправляет POST‑запрос на маршрут `"/cards/<int:card_id>/move"`.

### Шаг H5.1. View‑функция `card_move_view`

В `views_boards.py`, ниже `card_edit_view`, добавьте:

```python
def card_move_view(card_id: int):
    """Перенести карточку в другую колонку (в пределах одной доски)."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    conn = get_conn()

    # Загружаем карточку и связанную доску
    card = conn.execute(
        """
        SELECT
            c.id,
            c.column_id,
            col.board_id
        FROM cards c
        JOIN columns col ON c.column_id = col.id
        WHERE c.id = ?
        """,
        (card_id,),
    ).fetchone()

    if card is None:
        conn.close()
        abort(404)

    board_id = card["board_id"]

    if not can_edit_board(board_id):
        conn.close()
        abort(404)

    current_column_id = card["column_id"]
    new_column_id = request.form.get("new_column_id")

    try:
        new_column_id = int(new_column_id)
    except (TypeError, ValueError):
        conn.close()
        return redirect(url_for("board_view", board_id=board_id))

    if new_column_id == current_column_id:
        conn.close()
        return redirect(url_for("board_view", board_id=board_id))

    # Проверяем, что новая колонка принадлежит той же доске
    new_column = conn.execute(
        "SELECT id, board_id, wip_limit FROM columns WHERE id = ?",
        (new_column_id,),
    ).fetchone()

    if new_column is None or new_column["board_id"] != board_id:
        conn.close()
        abort(404)

    # Если у целевой колонки есть WIP‑лимит и карточка ещё не там — проверяем количество
    if new_column["wip_limit"] is not None and new_column["wip_limit"] > 0:
        current_count = count_cards_in_column(conn, new_column_id)
        if current_count >= new_column["wip_limit"]:
            conn.close()
            # Позже можно заменить на flash с текстом "Target column WIP limit reached"
            return redirect(url_for("board_view", board_id=board_id))

    # Помещаем карточку в конец списка целевой колонки
    row = conn.execute(
        "SELECT COALESCE(MAX(position), 0) AS max_pos FROM cards WHERE column_id = ?",
        (new_column_id,),
    ).fetchone()
    next_position = (row["max_pos"] or 0) + 1

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

### Шаг H5.2. Маршрут перемещения карточки в `app.py`

В `app.py` обновите импорт:

```python
from views_boards import (
    board_new_view,
    board_create_view,
    board_view_view,
    column_create_view,
    card_create_view,
    card_edit_view,
    card_move_view,
)
```

И ниже маршрута `card_edit` добавьте:

```python
@app.post("/cards/<int:card_id>/move")
def card_move(card_id):
    return card_move_view(card_id)
```

---

## H6. Обновляем шаблон `board.html`: карточки и формы

Теперь, когда логика готова, нужно обновить `board.html`, чтобы:

- внутри каждой колонки отображался список карточек;
- была форма создания новой карточки;
- у каждой карточки были формы редактирования и перемещения.

Напомним: в Части G внутри колонки был только текст‑заглушка.

### Шаг H6.1. Список карточек в колонке

В `board.html` найдите место, где отрисовывается тело колонки:

```html
            <div class="column-body">
              <p class="muted">
                Здесь позже появятся карточки этой колонки.
              </p>
            </div>
```

Замените его на:

```html
            <div class="column-body">
              {% set cards = cards_by_column.get(col.id, []) %}

              {% if not cards %}
                <p class="muted">
                  В этой колонке пока нет карточек.
                </p>
              {% else %}
                <ul class="card-list">
                  {% for card in cards %}
                    <li class="card-item">
                      <h4>{{ card.title }}</h4>
                      {% if card.description %}
                        <p>{{ card.description }}</p>
                      {% endif %}

                      {% if can_edit %}
                        <details class="card-edit">
                          <summary>Редактировать</summary>
                          <form method="post" action="{{ url_for('card_edit', card_id=card.id) }}" class="form">
                            <div class="form-group">
                              <label>Заголовок</label>
                              <input type="text" name="title" value="{{ card.title }}" required />
                            </div>
                            <div class="form-group">
                              <label>Описание</label>
                              <textarea name="description" rows="3">{{ card.description }}</textarea>
                            </div>
                            <button type="submit" class="btn btn-secondary">Сохранить</button>
                          </form>
                        </details>

                        <form method="post" action="{{ url_for('card_move', card_id=card.id) }}" class="form-inline card-move-form">
                          <label>Перенести в колонку:</label>
                          <select name="new_column_id">
                            {% for target in columns %}
                              <option value="{{ target.id }}" {% if target.id == col.id %}selected{% endif %}>
                                {{ target.name }}
                              </option>
                            {% endfor %}
                          </select>
                          <button type="submit" class="btn btn-secondary">Перенести</button>
                        </form>
                      {% endif %}
                    </li>
                  {% endfor %}
                </ul>
              {% endif %}
            </div>
```

Здесь:

- мы берём список карточек текущей колонки через `cards_by_column.get(col.id, [])`;
- если карточек нет — показываем текст «пока нет карточек»;
- если есть — выводим их списком, показывая заголовок и описание;
- для пользователей с правами редактирования:
  - отображаем компактную секцию `<details>` с формой редактирования карточки;
  - добавляем форму перемещения карточки в другую колонку.

### Шаг H6.2. Форма добавления карточки в колонку

Сразу **после** блока `column-body` (внутри `div.column`) добавьте форму добавления карточки, показываемую только при `can_edit`:

```html
            {% if can_edit %}
              <div class="card-create">
                <h4>Добавить карточку</h4>
                <form method="post" action="{{ url_for('card_create', column_id=col.id) }}" class="form">
                  <div class="form-group">
                    <label>Заголовок</label>
                    <input type="text" name="title" required />
                  </div>
                  <div class="form-group">
                    <label>Описание (необязательно)</label>
                    <textarea name="description" rows="3"></textarea>
                  </div>
                  <button type="submit" class="btn btn-primary">Создать</button>
                </form>
              </div>
            {% endif %}
```

Теперь каждая колонка:

- показывает свои карточки;
- даёт возможность создать новую карточку (если у пользователя есть права редактирования).

---

## H7. Подсчёт WIP‑лимита в заголовке колонки (опционально)

Сейчас заголовок колонки показывает «WIP: 0 / N», независимо от количества карточек. Если вы хотите уже на этом этапе считать текущее количество карточек, можно обновить шаблон так, чтобы вместо `0` подставлялось реальное количество:

В `board.html` найдите строку:

```html
                  WIP: 0 / {{ col.wip_limit }}
```

и замените её на:

```html
                  {% set current_cards = cards_by_column.get(col.id, []) %}
                  WIP: {{ current_cards|length }} / {{ col.wip_limit }}
```

Это не влияет на логику WIP‑ограничений (они уже реализованы в обработчиках), но делает отображение более точным и наглядным.

---

## H8. Проверка работы карточек и WIP

После всех изменений перезапустите приложение и выполните следующие действия:

1. **Создайте доску и несколько колонок.**
   - Добавьте хотя бы две колонки: например, «To Do» без WIP и «В работе» с WIP‑лимитом 2.
2. **Создайте несколько карточек в первой колонке.**
   - В каждой колонке должна появиться форма «Добавить карточку».
   - Заполните заголовок (и при желании описание), нажмите «Создать».
   - Новая карточка появляется внизу списка.
3. **Отредактируйте карточку.**
   - Нажмите «Редактировать» под карточкой, измените текст, сохраните.
   - После перезагрузки страницы изменения должны быть видны.
4. **Переместите карточку во вторую колонку.**
   - В форме «Перенести в колонку» выберите другую колонку и нажмите «Перенести».
   - Карточка должна исчезнуть из первой колонки и появиться во второй.
5. **Проверьте WIP‑лимит.**
   - Если у колонке установлен WIP‑лимит (например, 2) и в ней уже 2 карточки:
     - попытка добавить в неё ещё одну карточку должна быть заблокирована (карточка не появляется);
     - попытка перенести карточку из другой колонки в переполненную также должна быть заблокирована.
   - На этом шаге мы просто перенаправляем обратно на страницу доски; в Части J можно будет добавить явные сообщения (через `flash`) и более подробный журнал событий.

Если всё описанное выше работает, Часть H успешно выполнена.

---

## Итог Части H (Карточки)

После выполнения этой части у вас есть:

- таблица `cards` в базе данных, связанная с `columns` и `users`;
- хелпер `cards_for_board(board_id)` в `views_boards.py`, который группирует карточки по колонкам;
- вспомогательная функция `count_cards_in_column(conn, column_id)` для проверки WIP‑лимитов;
- обработчики:
  - `card_create_view(column_id)` — создание карточки в колонке с учётом WIP;
  - `card_edit_view(card_id)` — редактирование заголовка и описания карточки;
  - `card_move_view(card_id)` — перемещение карточки между колонками одной доски с проверкой WIP;
- маршруты в `app.py`:
  - `POST /columns/<column_id>/cards/new` → `card_create_view`;
  - `POST /cards/<card_id>/edit` → `card_edit_view`;
  - `POST /cards/<card_id>/move` → `card_move_view`;
- обновлённый шаблон `board.html`, который:
  - показывает карточки внутри колонок,
  - даёт форму добавления карточки,
  - даёт формы редактирования и перемещения карточек (для пользователей с правами редактирования),
  - может (опционально) отображать реальное значение `WIP: n / limit`.

Это завершает реализацию **карточек** и базовых WIP‑ограничений. В Части I вы добавите **совместную работу**: роли «editor» / «viewer» на доске и список досок, в которых пользователь участвует как приглашённый.
{% endraw %}

