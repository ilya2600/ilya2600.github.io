{% raw %}
# Часть G: Колонки на доске Kanban

В Части F вы реализовали первую сущность Kanban‑проекта — **доску** (`board`):

- таблица `boards` в базе данных;
- хелперы прав доступа к доске (`get_board_role`, `can_view_board`, `can_edit_board`, `is_board_owner`);
- модуль `views_boards.py` с логикой создания и просмотра доски;
- маршруты `/dashboard`, `/boards/new`, `/boards/<id>`;
- шаблоны `dashboard.html`, `board_new.html`, `board.html` (с заглушкой для колонок и карточек).

В этой части вы добавите **колонки** внутри доски. Карточки появятся только в следующих частях — сейчас мы готовим для них «рамку».

---

## Цель Части G

После выполнения этой части у вас будет:

- таблица `columns` в базе данных, связанная с таблицей `boards`;
- возможность добавлять на доску колонки с названием и опциональным WIP‑лимитом;
- обновлённое представление доски: список колонок по порядку;
- задел под ограничение WIP: для каждой колонки показывается «WIP: 0 / N» (0 до появления карточек в Части H).

**Видимое / проверяемое поведение:**

- откройте существующую доску → на ней есть форма «Добавить колонку»;
- заполните название (и при желании WIP‑лимит) → после отправки формы колонка появляется на доске;
- если вы указали, например, лимит `5` → в заголовке колонки отображается «WIP: 0 / 5».

---

## G1. Таблица `columns` в инициализации базы

Сначала нужно добавить в базу данных таблицу `columns`. Она будет хранить:

- к какой доске относится колонка;
- её название;
- порядок слева направо;
- опциональный WIP‑лимит (максимальное число карточек; пока только отображаем).

### Шаг G1.1. Открываем `init_db` и находим создание `boards`

Откройте файл `db.py` и найдите функцию:

```python
def init_db():
    ...
```

Внутри неё уже должны создаваться таблицы `settings`, `users` и `boards`. Нас интересует место сразу после блока:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS boards (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            owner_id INTEGER NOT NULL,
            title TEXT NOT NULL,
            description TEXT NOT NULL,
            is_archived INTEGER NOT NULL DEFAULT 0,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            FOREIGN KEY (owner_id) REFERENCES users(id)
        )
    """)
```

### Шаг G1.2. Добавляем `CREATE TABLE IF NOT EXISTS columns`

Сразу **после** создания таблицы `boards` добавьте блок:

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

Пояснения:

- `board_id` — доска, к которой относится колонка;
- `name` — название колонки (например, «To Do», «В работе», «Готово»);
- `position` — целое число для сортировки слева направо (`ORDER BY position`);
- `wip_limit` — опциональный WIP‑лимит:
  - `NULL` или `0` будет означать «лимит не задан»;
  - в Части H вы будете использовать это поле, чтобы ограничивать количество карточек;
- внешний ключ на `boards(id)` пока **не требует дополнительных действий**:
  - в Части J, где появится удаление досок и каскадное удаление, вы вернётесь к этому месту и при необходимости дополните схему / включите `PRAGMA foreign_keys = ON`.

В конце `init_db()` по‑прежнему должны вызываться `conn.commit()` и `conn.close()`.

**Проверка:** удалите файл `database.db` (если не жалко существующие данные), запустите приложение, затем откройте БД любым просмотрщиком SQLite и убедитесь, что таблица `columns` появилась со всеми полями.

> Если вы не хотите удалять `database.db`, можно временно открыть БД и выполнить `DROP TABLE IF EXISTS columns;` перед перезапуском приложения — тогда новая схема будет создана поверх.

---

## G2. Подготовка `views_boards.py` к работе с колонками

Теперь доработаем модуль `views_boards.py`, чтобы:

- загружать список колонок для доски;
- передавать их в шаблон `board.html`;
- разрешить добавление колонок только тем, у кого есть права редактирования доски.

### Шаг G2.1. Импортируем `can_edit_board`

Откройте `views_boards.py` и найдите блок импортов в начале файла:

```python
from flask import render_template, redirect, url_for, request, abort

from db import get_conn
from auth_utils import is_logged_in, current_user, can_view_board, is_board_owner
```

Замените последнюю строку на вариант с импортом `can_edit_board`:

```python
from auth_utils import is_logged_in, current_user, can_view_board, can_edit_board, is_board_owner
```

Функция `can_edit_board` уже реализована в `auth_utils.py` в Части F и возвращает `True` для владельца доски (и позже — для редакторов).

### Шаг G2.2. Хелпер `columns_for_board`

Под функцией `boards_for_user` добавьте новый хелпер:

```python
def columns_for_board(board_id: int):
    """Вернуть колонки доски по порядку."""
    conn = get_conn()
    rows = conn.execute(
        """
        SELECT id, board_id, name, position, wip_limit
        FROM columns
        WHERE board_id = ?
        ORDER BY position ASC, id ASC
        """,
        (board_id,),
    ).fetchall()
    conn.close()

    return rows
```

Здесь:

- мы выбираем все колонки заданной доски;
- сортируем по `position` (и дополнительно по `id`, чтобы порядок был стабильным при одинаковых позициях);
- возвращаем список строк `sqlite3.Row`, который удобно использовать в шаблоне как `col.name`, `col.wip_limit` и т.п.

### Шаг G2.3. Дополняем `board_view_view` колонками и флагом `can_edit`

Найдите существующую реализацию `board_view_view` в `views_boards.py`:

```python
def board_view_view(board_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    if not can_view_board(board_id):
        abort(404)

    conn = get_conn()
    board = conn.execute(
        """
        SELECT
            b.id,
            b.owner_id,
            b.title,
            b.description,
            b.is_archived,
            b.created_at,
            u.username AS owner_username
        FROM boards b
        JOIN users u ON b.owner_id = u.id
        WHERE b.id = ?
        """,
        (board_id,),
    ).fetchone()
    conn.close()

    if board is None:
        abort(404)

    return render_template(
        "board.html",
        board=board,
        is_owner=is_board_owner(board_id),
    )
```

Измените «хвост» функции так, чтобы:

- после загрузки доски был выполнен запрос за колонками;
- в шаблон передавались `columns` и флаг `can_edit`.

Итоговая часть функции должна выглядеть так (фрагмент начиная с `if board is None`):

```python
    if board is None:
        abort(404)

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

Теперь шаблон `board.html` будет знать:

- саму доску (`board`);
- список её колонок (`columns`);
- флаг `can_edit` — можно ли этой сессии показывать формы редактирования;
- флаг `is_owner` (пока не используется в этой части, но пригодится позже).

---

## G3. Обработчик добавления колонки

Следующая задача — дать пользователю возможность добавлять колонки на доску.

С точки зрения интерфейса это будет:

- форма на странице доски (в `board.html`);
- POST‑запрос на новый маршрут `"/boards/<int:board_id>/columns/new"`;
- перенаправление обратно на страницу доски после успешного добавления.

### Шаг G3.1. View‑функция `column_create_view` в `views_boards.py`

В том же файле `views_boards.py`, ниже `board_view_view`, добавьте функцию:

```python
def column_create_view(board_id: int):
    """Создать новую колонку на доске."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    # Проверяем, можно ли редактировать именно эту доску
    if not can_edit_board(board_id):
        abort(404)

    name = (request.form.get("name") or "").strip()
    wip_raw = (request.form.get("wip_limit") or "").strip()

    if not name:
        # Простейший вариант: вернуться на доску с флэш‑сообщением можно будет добавить позже.
        # Здесь достаточно не создавать колонку, если поле пустое.
        return redirect(url_for("board_view", board_id=board_id))

    # Преобразуем WIP‑лимит: пустая строка → None; некорректное число → игнорируем и считаем, что лимита нет.
    wip_limit = None
    if wip_raw:
        try:
            value = int(wip_raw)
            if value > 0:
                wip_limit = value
        except ValueError:
            pass

    conn = get_conn()

    # Находим следующий position: MAX(position) + 1 для этой доски.
    row = conn.execute(
        "SELECT COALESCE(MAX(position), 0) AS max_pos FROM columns WHERE board_id = ?",
        (board_id,),
    ).fetchone()
    next_position = (row["max_pos"] or 0) + 1

    conn.execute(
        """
        INSERT INTO columns (board_id, name, position, wip_limit)
        VALUES (?, ?, ?, ?)
        """,
        (board_id, name, next_position, wip_limit),
    )
    conn.commit()
    conn.close()

    return redirect(url_for("board_view", board_id=board_id))
```

Комментарии по логике:

- доска должна существовать и быть доступна для редактирования:
  - это проверяется через `can_edit_board(board_id)` (пока только владелец);
- `name` — обязательное поле;
- `wip_limit`:
  - пустая строка → нет лимита;
  - `0` или отрицательное значение → игнорируем (считаем, что лимита нет);
  - положительное целое → записываем в поле `wip_limit`;
- `position` выбирается как `MAX(position) + 1` для данной доски:
  - это обеспечивает добавление новых колонок «в конец»;
  - в будущих частях вы сможете реализовать изменение порядка колонок, если понадобится.

### Шаг G3.2. Подключаем маршрут в `app.py`

Теперь надо привязать эту view‑функцию к URL.

1. Откройте `app.py` и найдите импорт `views_boards`:

   ```python
   from views_boards import (
       board_new_view,
       board_create_view,
       board_view_view,
   )
   ```

2. Дополните его `column_create_view`:

   ```python
   from views_boards import (
       board_new_view,
       board_create_view,
       board_view_view,
       column_create_view,
   )
   ```

3. Ниже существующих маршрутов для досок добавьте новый маршрут для создания колонки:

   ```python
   @app.post("/boards/<int:board_id>/columns/new")
   def column_create(board_id):
       return column_create_view(board_id)
   ```

Важно:

- как и в остальных местах, сам маршрут остаётся «тонким» и просто делегирует работу в `views_boards.py`;
- для неавторизованных пользователей и тех, у кого нет прав редактирования доски, обработчик вернёт `404` (через `abort(404)` в `column_create_view`).

---

## G4. Обновляем шаблон `board.html`: список колонок

На данный момент `board.html` содержит заголовок доски, её описание и заглушку:

```html
    <h2>Колонки и карточки</h2>
    <p class="muted">
      Здесь позже появятся колонки и карточки Kanban‑доски.
    </p>
```

Пора заменить заглушку реальным списком колонок и формой «Добавить колонку».

### Шаг G4.1. Отрисовка колонок

Откройте `templates/board.html` и найдите блок «Колонки и карточки». Замените его на:

```html
    <h2>Колонки и карточки</h2>

    {% if not columns %}
      <p class="muted">
        На этой доске пока нет ни одной колонки.
      </p>
    {% else %}
      <div class="columns-row">
        {% for col in columns %}
          <div class="column">
            <header class="column-header">
              <strong>{{ col.name }}</strong>
              {% if col.wip_limit %}
                <span class="column-wip">
                  WIP: 0 / {{ col.wip_limit }}
                </span>
              {% endif %}
            </header>

            <div class="column-body">
              <p class="muted">
                Здесь позже появятся карточки этой колонки.
              </p>
            </div>
          </div>
        {% endfor %}
      </div>
    {% endif %}
```

Здесь:

- мы используем список `columns`, который передаётся из `board_view_view`;
- для каждой колонки отображаем:
  - название,
  - опциональный WIP‑лимит;
- пока карточек нет, поэтому в теле колонки добавлен текст‑заглушка:
  - в Части H он будет заменён на реальные карточки и формы.

> В этой части **число карточек всегда показывается как `0`**, потому что таблицы `cards` ещё нет. Это осознанное упрощение: в Части H вы добавите таблицу `cards` и замените «0» на реальный подсчёт количества карточек в колонке.

### Шаг G4.2. Форма «Добавить колонку»

Сразу **после** блока с колонками (всё, что вы вставили в G4.1) добавьте форму добавления новой колонки. Оборачиваем её проверкой `can_edit`, чтобы остальные пользователи не видели форму редактирования:

```html
    {% if can_edit %}
      <section class="add-column">
        <h3>Добавить колонку</h3>
        <form method="post" action="{{ url_for('column_create', board_id=board.id) }}" class="form-inline">
          <div class="form-group">
            <label for="col-name">Название</label>
            <input type="text" id="col-name" name="name" required />
          </div>

          <div class="form-group">
            <label for="col-wip">WIP‑лимит (необязательно)</label>
            <input type="number" id="col-wip" name="wip_limit" min="1" />
          </div>

          <button type="submit" class="btn btn-primary">Создать колонку</button>
        </form>
      </section>
    {% endif %}
```

Поведение:

- владелец доски (и в будущем редактор) увидит форму и сможет создавать колонки;
- гость и не имеющий доступа пользователь даже не увидят форму, потому что `can_edit` будет `False`.

Если у вас в `base.html` уже есть какие‑то базовые стили (`.form`, `.form-group`, `.btn`, `.card` и т.п.), форма будет выглядеть достаточно аккуратно. Внешний вид можно доработать по вкусу, главное — не ломать логику.

---

## G5. Проверка работы колонок

После внесения всех изменений перезапустите приложение и выполните следующую последовательность:

1. **Войдите под владельцем доски.**
   - Создайте новую доску или используйте уже существующую.
2. **Откройте страницу доски `/boards/<id>`.**
   - Должен отображаться заголовок доски и блок «Колонки и карточки».
   - Внизу блока (после списка колонок) должна быть форма «Добавить колонку».
3. **Добавьте колонку без WIP‑лимита.**
   - Введите название (например, «To Do»), оставьте поле WIP‑лимита пустым, нажмите кнопку.
   - Страница перезагрузится, и вы увидите новую колонку с названием и без строки `WIP: ...`.
4. **Добавьте колонку с WIP‑лимитом.**
   - Введите другое название (например, «В работе»), в поле WIP‑лимита укажите, например, `5`.
   - После отправки формы рядом с названием колонки должно отображаться «WIP: 0 / 5».
5. **Попробуйте открыть эту же доску без входа.**
   - Выйдите из аккаунта и попробуйте открыть `/boards/<id>` напрямую.
   - Должно сработать ограничение из Части F: неавторизованный пользователь будет перенаправлен на страницу логина.

Если всё работает так, как описано, — Часть G выполнена.

---

## Итог Части G (Колонки)

После выполнения этой части у вас есть:

- таблица `columns` в базе данных, связанная с `boards`;
- хелпер `columns_for_board(board_id)` в `views_boards.py`;
- обновлённая функция `board_view_view`, которая:
  - проверяет права доступа,
  - загружает доску и её колонки,
  - передаёт в шаблон `board`, `columns`, `can_edit`, `is_owner`;
- новая view‑функция `column_create_view(board_id)` для создания колонок;
- маршрут `POST /boards/<board_id>/columns/new` в `app.py`;
- обновлённый шаблон `board.html`:
  - выводит список колонок по порядку,
  - отображает WIP‑лимит как «WIP: 0 / N»,
  - показывает форму «Добавить колонку» только тем, кто может редактировать доску.

Это завершает базовую поддержку **колонок**. В Части H вы добавите **карточки** внутри колонок, реализуете добавление, редактирование и перемещение карточек, а также начнёте реально применять WIP‑лимиты.
{% endraw %}

