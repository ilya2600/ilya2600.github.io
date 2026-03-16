{% raw %}
# Часть I: Совместная работа — роли доски и приглашённые участники

В Частях F–H вы реализовали:

- **доски** (`boards`) и права доступа владельца;
- **колонки** (`columns`) с опциональными WIP‑лимитами;
- **карточки** (`cards`) внутри колонок, с возможностью создавать, редактировать и перемещать их, а также базовой проверкой WIP‑лимита.

Пока что редактировать доску может только её владелец. В этой части вы добавите **совместную работу**:

- приглашённых участников доски с ролями:
  - `viewer` — только просмотр,
  - `editor` — может редактировать доску почти как владелец (добавлять / изменять колонки и карточки);
- обновлённую логику прав доступа к доске (`get_board_role`, `can_view_board`, `can_edit_board`);
- новый раздел на дашборде: **«Доски, в которых я участвую»**.

---

## Цель Части I

После выполнения этой части у вас будет:

- таблица `collaborators` в базе данных, связывающая пользователей и доски;
- доработанные функции прав доступа:
  - `get_board_role(board_id)` возвращает одно из значений:
    - `"owner"`, `"editor"`, `"viewer"` или `None`;
  - `can_view_board(board_id)` и `can_edit_board(board_id)` используют новые роли;
- на странице доски владелец увидит раздел **«Участники»**, где:
  - может добавить пользователя как `viewer` или `editor`,
  - может удалить участника;
- на дашборде появится блок **«Доски, в которых я участвую»**, в котором:
  - перечислены доски, где пользователь добавлен в `collaborators`.

**Видимое / проверяемое поведение:**

- владелец доски может:
  - пригласить другого пользователя как редактора → тот видит все формы редактирования и может менять колонки / карточки;
  - пригласить как зрителя → тот видит доску, но не видит форм редактирования;
- приглашённый пользователь:
  - видит доску в разделе «Доски, в которых я участвую» на дашборде;
  - после удаления из участников теряет доступ к доске.

---

## I1. Таблица `collaborators` в инициализации базы

Сначала добавим таблицу, которая хранит участников досок.

### Шаг I1.1. Открываем `init_db` и находим конец списка таблиц

Откройте файл `db.py` и найдите функцию:

```python
def init_db():
    ...
```

Внутри неё уже создаются таблицы:

- `settings`,
- `users`,
- `boards`,
- `columns`,
- `cards`.

Нас интересует место **после** создания таблицы `cards`:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS cards (
            ...
        )
    """)
```

### Шаг I1.2. Добавляем `CREATE TABLE IF NOT EXISTS collaborators`

Сразу после блока `cards` добавьте:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS collaborators (
            board_id INTEGER NOT NULL,
            user_id INTEGER NOT NULL,
            role TEXT NOT NULL CHECK (role IN ('viewer', 'editor')),
            PRIMARY KEY (board_id, user_id),
            FOREIGN KEY (board_id) REFERENCES boards(id),
            FOREIGN KEY (user_id) REFERENCES users(id)
        )
    """)
```

Пояснения:

- `board_id`, `user_id` — определяют пару «доска–пользователь»;
- `role` — роль этого пользователя на доске:
  - `"viewer"` — может только просматривать;
  - `"editor"` — может редактировать;
- `PRIMARY KEY (board_id, user_id)` гарантирует, что одна и та же пара не появится в таблице дважды;
- внешние ключи пока, как и раньше, **без каскада**:
  - вопросы каскадного удаления будут решены в Части J.

В конце `init_db()` по‑прежнему должны вызываться `conn.commit()` и `conn.close()`.

---

## I2. Расширяем `get_board_role` и права доступа

Теперь доработаем функции в `auth_utils.py`, чтобы они учитывали таблицу `collaborators`.

### Шаг I2.1. Обновляем `get_board_role`

Откройте `auth_utils.py` и найдите текущую реализацию:

```python
def get_board_role(board_id):
    """Вернуть роль текущего пользователя для доски: 'owner' или None."""
    user = current_user()
    if user is None:
        return None

    conn = get_conn()
    board = conn.execute(
        "SELECT owner_id FROM boards WHERE id = ?",
        (board_id,),
    ).fetchone()
    conn.close()

    if board is None:
        return None

    if board["owner_id"] == user["id"]:
        return "owner"

    # Позже сюда можно будет добавить проверки для приглашённых участников
    return None
```

Замените её на вариант, который учитывает таблицу `collaborators`:

```python
def get_board_role(board_id):
    """Вернуть роль текущего пользователя для доски: 'owner', 'editor', 'viewer' или None."""
    user = current_user()
    if user is None:
        return None

    conn = get_conn()

    # Сначала проверяем, не является ли пользователь владельцем доски
    board = conn.execute(
        "SELECT owner_id FROM boards WHERE id = ?",
        (board_id,),
    ).fetchone()

    if board is None:
        conn.close()
        return None

    if board["owner_id"] == user["id"]:
        conn.close()
        return "owner"

    # Если не владелец — ищем запись в collaborators
    collab = conn.execute(
        """
        SELECT role
        FROM collaborators
        WHERE board_id = ? AND user_id = ?
        """,
        (board_id, user["id"]),
    ).fetchone()

    conn.close()

    if collab is None:
        return None

    role = collab["role"]
    if role in ("viewer", "editor"):
        return role

    return None
```

### Шаг I2.2. Обновляем `can_view_board` и `can_edit_board`

Чуть ниже находятся функции:

```python
def can_view_board(board_id):
    """Можно ли просматривать эту доску."""
    return get_board_role(board_id) is not None


def can_edit_board(board_id):
    """Можно ли редактировать доску (менять название, добавлять колонки и карточки)."""
    role = get_board_role(board_id)
    return role == "owner"
```

`can_view_board` уже подходит под новую логику: любую ненулевую роль (`owner`, `editor`, `viewer`) будет считать «просмотр разрешён». `can_edit_board` же нужно расширить, чтобы давать права редактирования не только владельцу, но и редактору.

Замените `can_edit_board` на:

```python
def can_edit_board(board_id):
    """Можно ли редактировать доску (менять название, добавлять колонки и карточки)."""
    role = get_board_role(board_id)
    return role in ("owner", "editor")
```

Функция `is_board_owner` можно оставить без изменений — она по‑прежнему проверяет только владельца.

> Важно: мы сознательно **не даём** права редактирования зрителям (`viewer`). Они видят доску, но не могут менять её содержимое.

---

## I3. Раздел «Участники» на странице доски

Теперь добавим на страницу доски:

- список приглашённых участников,
- форму для добавления нового участника,
- кнопку для удаления участника.

Логику реализуем во `views_boards.py`, но использовать будем права, описанные в `auth_utils.py`.

### Шаг I3.1. Хелпер для получения списка участников

Откройте `views_boards.py`. Рядом с существующими хелперами (например, `boards_for_user`, `columns_for_board`, `cards_for_board`) добавьте функцию:

```python
def collaborators_for_board(board_id: int):
    """Вернуть всех участников доски с именами и ролями."""
    conn = get_conn()
    rows = conn.execute(
        """
        SELECT
            c.board_id,
            c.user_id,
            c.role,
            u.username
        FROM collaborators c
        JOIN users u ON c.user_id = u.id
        WHERE c.board_id = ?
        ORDER BY u.username ASC
        """,
        (board_id,),
    ).fetchall()
    conn.close()

    return rows
```

### Шаг I3.2. Подгружаем участников в `board_view_view`

`board_view_view` уже загружает доску, колонки и карточки. Найдите её реализацию и дополните:

```python
    columns = columns_for_board(board_id)
    cards_by_column = cards_for_board(board_id)
    can_edit = can_edit_board(board_id)

    collaborators = collaborators_for_board(board_id)

    return render_template(
        "board.html",
        board=board,
        columns=columns,
        cards_by_column=cards_by_column,
        collaborators=collaborators,
        is_owner=is_board_owner(board_id),
        can_edit=can_edit,
    )
```

Теперь шаблон `board.html` будет получать список участников доски.

### Шаг I3.3. View‑функции для добавления и удаления участника

Во `views_boards.py`, ниже существующих view‑функций (например, под `card_move_view`), добавьте ещё две:

```python
def collaborator_add_view(board_id: int):
    """Добавить участника к доске (только владелец)."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    # Только владелец может управлять участниками
    if not is_board_owner(board_id):
        abort(404)

    user_id_raw = request.form.get("user_id")
    role = (request.form.get("role") or "").strip()

    try:
        user_id = int(user_id_raw)
    except (TypeError, ValueError):
        return redirect(url_for("board_view", board_id=board_id))

    if role not in ("viewer", "editor"):
        return redirect(url_for("board_view", board_id=board_id))

    conn = get_conn()

    # Нельзя добавить владельца как участника
    board = conn.execute(
        "SELECT owner_id FROM boards WHERE id = ?",
        (board_id,),
    ).fetchone()

    if board is None:
        conn.close()
        abort(404)

    if board["owner_id"] == user_id:
        conn.close()
        return redirect(url_for("board_view", board_id=board_id))

    # Проверяем, что пользователь существует и не заархивирован
    user = conn.execute(
        "SELECT id FROM users WHERE id = ? AND archived_at IS NULL",
        (user_id,),
    ).fetchone()

    if user is None:
        conn.close()
        return redirect(url_for("board_view", board_id=board_id))

    # Добавляем участника, если его ещё нет
    conn.execute(
        """
        INSERT OR IGNORE INTO collaborators (board_id, user_id, role)
        VALUES (?, ?, ?)
        """,
        (board_id, user_id, role),
    )
    conn.commit()
    conn.close()

    return redirect(url_for("board_view", board_id=board_id))


def collaborator_remove_view(board_id: int):
    """Удалить участника с доски (только владелец)."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    if not is_board_owner(board_id):
        abort(404)

    user_id_raw = request.form.get("user_id")
    try:
        user_id = int(user_id_raw)
    except (TypeError, ValueError):
        return redirect(url_for("board_view", board_id=board_id))

    conn = get_conn()
    conn.execute(
        "DELETE FROM collaborators WHERE board_id = ? AND user_id = ?",
        (board_id, user_id),
        )
    conn.commit()
    conn.close()

    return redirect(url_for("board_view", board_id=board_id))
```

Поведение:

- только владелец (`is_board_owner(board_id)`) может вызывать эти действия;
- **добавление**:
  - не даёт добавить самого владельца;
  - не даёт добавить несуществующего / заархивированного пользователя;
  - `INSERT OR IGNORE` предотвращает дублирование;
- **удаление**:
  - просто убирает запись из таблицы `collaborators`.

### Шаг I3.4. Маршруты для управления участниками в `app.py`

Откройте `app.py` и дополните импорт из `views_boards`:

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
)
```

Ниже существующих маршрутов доски добавьте:

```python
@app.post("/boards/<int:board_id>/collaborators/add")
def collaborator_add(board_id):
    return collaborator_add_view(board_id)


@app.post("/boards/<int:board_id>/collaborators/remove")
def collaborator_remove(board_id):
    return collaborator_remove_view(board_id)
```

---

## I4. Обновляем шаблон `board.html`: раздел «Участники»

Теперь нужно отобразить на странице доски список участников и форму добавления нового.

### Шаг I4.1. Секция участников (только для владельца)

Откройте `templates/board.html`. Внутри основного `<section class="card">...</section>` (например, после блока с описанием доски или после блока с колонками) добавьте новый раздел:

```html
    {% if is_owner %}
      <h2>Участники</h2>

      {% if not collaborators %}
        <p class="muted">Пока на этой доске нет приглашённых участников.</p>
      {% else %}
        <table class="table">
          <thead>
            <tr>
              <th>Пользователь</th>
              <th>Роль</th>
              <th>Действия</th>
            </tr>
          </thead>
          <tbody>
            {% for c in collaborators %}
              <tr>
                <td>{{ c.username }}</td>
                <td>
                  {% if c.role == 'editor' %}
                    Редактор
                  {% elif c.role == 'viewer' %}
                    Только просмотр
                  {% else %}
                    {{ c.role }}
                  {% endif %}
                </td>
                <td>
                  <form method="post" action="{{ url_for('collaborator_remove', board_id=board.id) }}" style="display: inline;">
                    <input type="hidden" name="user_id" value="{{ c.user_id }}" />
                    <button type="submit" class="btn btn-secondary">Удалить</button>
                  </form>
                </td>
              </tr>
            {% endfor %}
          </tbody>
        </table>
      {% endif %}

      <h3>Добавить участника</h3>

      {% if not available_users %}
        <p class="muted">Нет доступных пользователей для приглашения (все уже приглашены или это только вы).</p>
      {% else %}
        <form method="post" action="{{ url_for('collaborator_add', board_id=board.id) }}" class="form-inline">
          <div class="form-group">
            <label for="collab-user-id">Пользователь</label>
            <select id="collab-user-id" name="user_id">
              {% for u in available_users %}
                <option value="{{ u.id }}">{{ u.username }}</option>
              {% endfor %}
            </select>
          </div>

          <div class="form-group">
            <label for="collab-role">Роль</label>
            <select id="collab-role" name="role">
              <option value="viewer">Только просмотр</option>
              <option value="editor">Редактор</option>
            </select>
          </div>

          <button type="submit" class="btn btn-primary">Пригласить</button>
        </form>
      {% endif %}
    {% endif %}
```

Здесь:

- раздел «Участники» виден **только владельцу** (`is_owner`);
- пока для простоты приглашение происходит по `ID` пользователя (это уменьшает объём этой части; в будущем можно заменить на выпадающий список имён);
- таблица участников показывает:
  - имя пользователя,
  - его роль,
  - кнопку «Удалить».

> Важно: все действия для участников делаются через POST‑запросы и используют уже реализованные view‑функции.

---

## I5. Дорабатываем дашборд: «Доски, в которых я участвую»

Сейчас дашборд (`dashboard_view` в `views_auth.py` и `dashboard.html`) показывает только **собственные** доски пользователя. Добавим раздел с досками, в которых пользователь участвует как приглашённый.

### Шаг I5.1. Хелпер `collaborator_boards_for_user`

Во `views_boards.py`, рядом с `boards_for_user`, добавьте функцию:

```python
def collaborator_boards_for_user():
    """Вернуть активные доски, где текущий пользователь — приглашённый участник."""
    user = current_user()
    if user is None:
        return []

    conn = get_conn()
    rows = conn.execute(
        """
        SELECT
            b.id,
            b.title,
            b.description,
            b.is_archived,
            b.created_at,
            c.role AS collab_role
        FROM boards b
        JOIN collaborators c
            ON c.board_id = b.id
        WHERE
            c.user_id = ?
            AND b.is_archived = 0
        ORDER BY b.created_at DESC
        """,
        (user["id"],),
    ).fetchall()
    conn.close()

    return rows
```

### Шаг I5.2. Подключаем хелпер в `views_auth.dashboard_view`

Откройте `views_auth.py` и вверху файла, рядом с импортом `boards_for_user`, добавьте:

```python
from views_boards import boards_for_user, collaborator_boards_for_user
```

Найдите реализацию `dashboard_view`:

```python
def dashboard_view():
    if not is_logged_in():
        ...

    user = current_user()
    if user is None:
        ...

    boards = boards_for_user()
    return render_template("dashboard.html", user=user, boards=boards)
```

Замените её на вариант, который передаёт в шаблон и «чужие» доски:

```python
def dashboard_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    if user is None:
        session.clear()
        return redirect(url_for("login_form"))

    boards = boards_for_user()
    collaborator_boards = collaborator_boards_for_user()

    return render_template(
        "dashboard.html",
        user=user,
        boards=boards,
        collaborator_boards=collaborator_boards,
    )
```

### Шаг I5.3. Обновляем шаблон `dashboard.html`

Откройте `templates/dashboard.html`. Сейчас он показывает только список собственных досок:

```html
    <h1>Ваши доски</h1>
    ...
    {% if not boards %}
      <p class="muted">У вас пока нет ни одной доски.</p>
    {% else %}
      <table class="table">
        ...
      </table>
    {% endif %}
```

Сначала убедитесь, что этот блок остаётся как есть — он по‑прежнему отвечает за «мои» доски.

Затем **ниже** этого блока добавьте новый раздел для досок, в которых пользователь участвует:

```html
    <h2 style="margin-top: 32px;">Доски, в которых вы участвуете</h2>

    {% if not collaborator_boards %}
      <p class="muted">Вы пока не приглашены ни на одну доску других пользователей.</p>
    {% else %}
      <table class="table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Название</th>
            <th>Описание</th>
            <th>Роль</th>
          </tr>
        </thead>
        <tbody>
          {% for b in collaborator_boards %}
            <tr>
              <td><a href="{{ url_for('board_view', board_id=b.id) }}">{{ b.id }}</a></td>
              <td><a href="{{ url_for('board_view', board_id=b.id) }}">{{ b.title }}</a></td>
              <td>{{ b.description }}</td>
              <td>
                {% if b.collab_role == 'editor' %}
                  Редактор
                {% elif b.collab_role == 'viewer' %}
                  Только просмотр
                {% else %}
                  {{ b.collab_role }}
                {% endif %}
              </td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    {% endif %}
```

Теперь дашборд:

- в верхнем блоке показывает **ваши** доски (вы владелец);
- в нижнем — доски, на которые вас пригласили другие пользователи.

---

## I6. Проверка совместной работы и ролей

После внесения всех изменений перезапустите приложение и выполните проверку:

1. **Подготовьте двух пользователей.**
   - Например, `master` (администратор / владелец доски) и ещё один обычный пользователь.
2. **Создайте доску под владельцем.**
   - Залогиньтесь как владелец, создайте доску, откройте её страницу.
3. **Пригласите второго пользователя как редактора.**
   - В разделе «Участники» введите `ID` второго пользователя и выберите роль «Редактор».
   - Выйдите и войдите под вторым пользователем.
   - На дашборде во втором разделе должна появиться эта доска; при открытии вы должны видеть формы добавления колонок и карточек, а также формы редактирования и перемещения.
4. **Измените роль на зрителя.**
   - Залогиньтесь снова под владельцем, удалите участника и пригласите его заново, выбрав роль `viewer` в выпадающем списке.
   - Под вторым пользователем:
     - доска всё ещё отображается в разделе «Доски, в которых вы участвуете»;
     - но на странице доски не должно быть форм добавления / редактирования / перемещения (только просмотр).
5. **Удалите участника.**
   - Под владельцем нажмите «Удалить» напротив участника.
   - После этого:
     - под вторым пользователем доска должна исчезнуть из раздела «Доски, в которых вы участвуете»;
     - попытка открыть доску по прямой ссылке должна привести к ошибке 404 (или переадресации на логин, если не авторизован).

Если всё работает так, как описано, Часть I успешно выполнена.

---

## Итог Части I (Роли и участники)

После выполнения этой части у вас есть:

- таблица `collaborators` в базе данных, описывающая участников досок и их роли;
- расширенная логика прав доступа:
  - `get_board_role(board_id)` возвращает `'owner'`, `'editor'`, `'viewer'` или `None`,
  - `can_view_board(board_id)` — допускает все три роли,
  - `can_edit_board(board_id)` — допускает `'owner'` и `'editor'`;
- раздел «Участники» на странице доски:
  - владелец видит список приглашённых пользователей и их роли,
  - владелец может приглашать новых участников и удалять существующих;
- обновлённый дашборд:
  - «Ваши доски» (где вы владелец),
  - «Доски, в которых вы участвуете» (где вы приглашённый `viewer` или `editor`).

Теперь доски Kanban поддерживают базовую совместную работу. В Части J вы добавите:

- журнал активности карточек;
- архивацию, восстановление и удаление досок;
- поведение «только чтение» для архивных досок.
{% endraw %}

