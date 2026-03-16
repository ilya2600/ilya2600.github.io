{% raw %}
# Часть G: Участники каналов — членство, вступление и выход

В Части F вы добавили каналы: таблицу `channels`, дашборд со списком «моих» каналов (по владельцу) и публичных каналов, создание канала и просмотр одного канала. Кто именно считается участником канала и как пользователь вступает в канал или выходит из него, до сих пор не было смоделировано.

В этой части вы вводите **модель членства**:

- таблицу `channel_members` в `db.py` (связь пользователь–канал и роль в канале);
- при создании канала — автоматическое добавление создателя в участники с ролью владельца;
- дашборд: «мои каналы» — каналы, где пользователь есть в `channel_members` (а не только владелец по `owner_id`);
- вступление в **публичный** канал (кнопка «Вступить», добавление в `channel_members` с ролью по типу канала);
- выход из канала (удаление из `channel_members`; если выходит владелец — передача владения другому участнику или канал без владельца);
- доступ к просмотру канала: **приватный** канал видят только участники и админ; публичный могут открыть все, но для участия нужно вступить.

Управление участниками внутри канала (приглашения, удаление участников владельцем) остаётся на следующую часть.

---

## G1. Таблица `channel_members` в `db.py`

Участник канала — это запись «пользователь X в канале Y с ролью Z». Роли на уровне канала: `owner`, `member`, `read_only`. Владелец канала по смыслу совпадает с записью в `channel_members` с ролью `owner`; поле `channels.owner_id` удобно оставить для быстрого поиска владельца и для передачи владения при выходе.

### Шаг G1.1. Поля таблицы `channel_members`

Таблица хранит: идентификатор канала (`channel_id`), идентификатор пользователя (`user_id`), роль в канале (`role`: `owner` / `member` / `read_only`). Пара (канал, пользователь) должна быть уникальной — один пользователь не может быть в одном канале дважды.

### Шаг G1.2. Добавляем таблицу в `init_db()`

Откройте `db.py` и найдите функцию `init_db()`. Сразу **после** блока `CREATE TABLE IF NOT EXISTS channels (...)` добавьте создание таблицы `channel_members`:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS channel_members (
            channel_id INTEGER NOT NULL,
            user_id INTEGER NOT NULL,
            role TEXT NOT NULL CHECK (role IN ('owner', 'member', 'read_only')),
            PRIMARY KEY (channel_id, user_id),
            FOREIGN KEY (channel_id) REFERENCES channels(id),
            FOREIGN KEY (user_id) REFERENCES users(id)
        )
    """)
```

В итоге `init_db()` создаёт таблицы `settings`, `users`, `channels`, `channel_members`, затем выполняет `conn.commit()` и `conn.close()`.

---

## G2. Создание канала: добавление создателя в участники

При создании канала создатель должен сразу стать участником с ролью `owner`. Поле `channels.owner_id` уже устанавливается в `channel_create_view()`; нужно дополнительно вставить строку в `channel_members`.

### Шаг G2.1. Получение ID созданного канала и вставка в `channel_members`

В SQLite после `INSERT` можно получить ID последней вставленной строки через `cursor.lastrowid`. В коде с `conn.execute()` нужно использовать тот же курсор или выполнить `SELECT last_insert_rowid()`.

Откройте `views_channels.py` и найдите функцию `channel_create_view()`. После успешного `INSERT INTO channels` и **до** `conn.commit()` добавьте вставку в `channel_members` и получите `channel_id`:

1. Выполните вставку канала через курсор, чтобы использовать `lastrowid`:

```python
    user = current_user()
    conn = get_conn()
    try:
        cur = conn.execute(
            "INSERT INTO channels (name, type, owner_id) VALUES (?, ?, ?)",
            (name, ctype, user["id"]),
        )
        channel_id = cur.lastrowid
        conn.execute(
            "INSERT INTO channel_members (channel_id, user_id, role) VALUES (?, ?, 'owner')",
            (channel_id, user["id"]),
        )
        conn.commit()
        flash("Канал создан.")
    except sqlite3.IntegrityError:
        flash("Канал с таким именем уже существует.")
    conn.close()
```

Таким образом, при создании канала создатель автоматически попадает в `channel_members` с ролью `owner`. Поле `channels.owner_id` при этом совпадает с этим пользователем.

---

## G3. Дашборд: «мои каналы» по членству, публичные — для вступления

«Мои каналы» должны показывать все каналы, в которых текущий пользователь состоит (любая роль в `channel_members`), а не только каналы, где он владелец по `owner_id`. Список «доступных публичных каналов» — это публичные каналы, в которых пользователь **не** состоит; для них на дашборде будет кнопка «Вступить».

### Шаг G3.1. Выборка «моих» каналов по `channel_members`

В `views_channels.py` в функции `dashboard_view()` замените выборку `my_channels`. Нужны каналы, в которых есть текущий пользователь, и его роль в канале:

```python
    # Каналы, где пользователь — участник (по channel_members)
    my_channels = conn.execute("""
        SELECT c.id, c.name, c.type, c.owner_id, cm.role AS my_role
        FROM channels c
        INNER JOIN channel_members cm ON c.id = cm.channel_id
        WHERE cm.user_id = ?
        ORDER BY c.name
    """, (user["id"],)).fetchall()
```

Переменная `my_role` понадобится в шаблоне (например, для отображения «владелец» / «участник» / «только чтение»).

### Шаг G3.2. Выборка публичных каналов для вступления

Публичные каналы, к которым можно присоединиться — те, у которых `type = 'public'` и текущего пользователя **нет** в `channel_members`:

```python
    # Публичные каналы, в которых пользователь ещё не состоит
    public_joinable = conn.execute("""
        SELECT c.id, c.name, c.type
        FROM channels c
        WHERE c.type = 'public'
          AND NOT EXISTS (
              SELECT 1 FROM channel_members cm
              WHERE cm.channel_id = c.id AND cm.user_id = ?
          )
        ORDER BY c.name
    """, (user["id"],)).fetchall()
```

Оставшийся код `dashboard_view()` (проверка входа, `conn.close()`, `return render_template(...)`) не меняется, но в шаблон теперь передаётся `my_channels` с полем `my_role`. Итоговый фрагмент функции для справки:

```python
def dashboard_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    if user is None:
        session.clear()
        return redirect(url_for("login_form"))

    conn = get_conn()

    my_channels = conn.execute("""
        SELECT c.id, c.name, c.type, c.owner_id, cm.role AS my_role
        FROM channels c
        INNER JOIN channel_members cm ON c.id = cm.channel_id
        WHERE cm.user_id = ?
        ORDER BY c.name
    """, (user["id"],)).fetchall()

    public_joinable = conn.execute("""
        SELECT c.id, c.name, c.type
        FROM channels c
        WHERE c.type = 'public'
          AND NOT EXISTS (
              SELECT 1 FROM channel_members cm
              WHERE cm.channel_id = c.id AND cm.user_id = ?
          )
        ORDER BY c.name
    """, (user["id"],)).fetchall()

    conn.close()

    return render_template(
        "dashboard.html",
        user=user,
        my_channels=my_channels,
        public_joinable=public_joinable,
    )
```

---

## G4. Вступление в публичный канал (Join)

Пользователь может вступить в канал только если канал **публичный** и он ещё не участник. После вступления его роль в канале: `member`, если тип канала `public`, и `read_only`, если тип канала `read_only` (канал «только чтение» тоже может быть публичным в смысле видимости — тогда вступить можно, но с ролью только чтение).

### Шаг G4.1. Функция вступления в канал

В `views_channels.py` добавьте функцию, которая обрабатывает POST-запрос на вступление (её будет вызывать маршрут вида `POST /channels/<id>/join`):

```python
def channel_join_view(channel_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    conn = get_conn()

    ch = conn.execute(
        "SELECT id, name, type FROM channels WHERE id = ?",
        (channel_id,),
    ).fetchone()

    if ch is None:
        conn.close()
        abort(404)

    if ch["type"] != "public":
        conn.close()
        flash("Вступить можно только в публичный канал.")
        return redirect(url_for("dashboard"))

    existing = conn.execute(
        "SELECT 1 FROM channel_members WHERE channel_id = ? AND user_id = ?",
        (channel_id, user["id"]),
    ).fetchone()
    if existing:
        conn.close()
        flash("Вы уже в этом канале.")
        return redirect(url_for("channel_view", channel_id=channel_id))

    # Роль при вступлении: в канале типа read_only — read_only, иначе member
    role = "read_only" if ch["type"] == "read_only" else "member"
    conn.execute(
        "INSERT INTO channel_members (channel_id, user_id, role) VALUES (?, ?, ?)",
        (channel_id, user["id"], role),
    )
    conn.commit()
    conn.close()

    flash("Вы вступили в канал.")
    return redirect(url_for("channel_view", channel_id=channel_id))
```

Обратите внимание: проверяем тип канала по полю `ch["type"]`; для канала с типом `read_only` новый участник получает роль `read_only`, иначе — `member`. Приватные каналы в этой части не поддерживают самостоятельное вступление (только по приглашению в следующих частях).

### Шаг G4.2. Маршрут для вступления

В `app.py` добавьте импорт `channel_join_view` из `views_channels` и маршрут:

```python
from views_channels import (
    dashboard_view,
    channel_new_view,
    channel_create_view,
    channel_view_view,
    channel_join_view,
)
```

И маршрут (например, после маршрута просмотра канала):

```python
@app.post("/channels/<int:channel_id>/join")
def channel_join(channel_id):
    return channel_join_view(channel_id)
```

---

## G5. Выход из канала (Leave)

Участник может покинуть канал. При этом запись из `channel_members` удаляется. Если выходит **владелец** канала, нужно либо передать владение другому участнику (например, первому по id с ролью `member` или `read_only`), либо, если участников больше нет или передавать некому, оставить канал без владельца (`channels.owner_id = NULL`).

### Шаг G5.1. Логика выхода и передачи владения

В `views_channels.py` добавьте функцию:

```python
def channel_leave_view(channel_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    conn = get_conn()

    ch = conn.execute(
        "SELECT id, name, owner_id FROM channels WHERE id = ?",
        (channel_id,),
    ).fetchone()

    if ch is None:
        conn.close()
        abort(404)

    member = conn.execute(
        "SELECT role FROM channel_members WHERE channel_id = ? AND user_id = ?",
        (channel_id, user["id"]),
    ).fetchone()

    if member is None:
        conn.close()
        flash("Вы не состоите в этом канале.")
        return redirect(url_for("dashboard"))

    # Удаляем пользователя из участников
    conn.execute(
        "DELETE FROM channel_members WHERE channel_id = ? AND user_id = ?",
        (channel_id, user["id"]),
    )

    # Если выходил владелец — передаём владение или обнуляем owner_id
    if ch["owner_id"] == user["id"]:
        new_owner = conn.execute("""
            SELECT user_id FROM channel_members
            WHERE channel_id = ?
            ORDER BY role = 'member' DESC, user_id ASC
            LIMIT 1
        """, (channel_id,)).fetchone()

        if new_owner:
            conn.execute(
                "UPDATE channels SET owner_id = ? WHERE id = ?",
                (new_owner["user_id"], channel_id),
            )
            conn.execute(
                "UPDATE channel_members SET role = 'owner' WHERE channel_id = ? AND user_id = ?",
                (channel_id, new_owner["user_id"]),
            )
        else:
            conn.execute(
                "UPDATE channels SET owner_id = NULL WHERE id = ?",
                (channel_id,),
            )

    conn.commit()
    conn.close()

    flash("Вы вышли из канала.")
    return redirect(url_for("dashboard"))
```

Пояснение: после удаления текущего пользователя из `channel_members` проверяем, был ли он владельцем. Если да — ищем любого оставшегося участника (например, предпочитая роль `member`, затем по `user_id`). Если такой есть — ставим его в `channels.owner_id` и в `channel_members` даём ему роль `owner`. Если участников не осталось — обнуляем `owner_id`.

### Шаг G5.2. Маршрут для выхода

В `app.py` добавьте импорт `channel_leave_view` и маршрут:

```python
from views_channels import (
    dashboard_view,
    channel_new_view,
    channel_create_view,
    channel_view_view,
    channel_join_view,
    channel_leave_view,
)
```

```python
@app.post("/channels/<int:channel_id>/leave")
def channel_leave(channel_id):
    return channel_leave_view(channel_id)
```

---

## G6. Доступ к просмотру канала: только участники (и админ) для приватных

Сейчас любой вошедший пользователь может открыть любой канал по id. Нужно ограничить: **приватный** канал могут просматривать только те, кто есть в `channel_members`, плюс пользователи с глобальной ролью админа. Публичные каналы по-прежнему может открыть любой (на странице канала незарегистрированному в канале показывается возможность «Вступить»).

### Шаг G6.1. Проверка доступа в `channel_view_view`

В `views_channels.py` в функции `channel_view_view(channel_id)` после загрузки канала `ch` и проверки `if ch is None: abort(404)` добавьте проверку доступа. Нужен импорт `is_admin` из `auth_utils`:

```python
from auth_utils import is_logged_in, current_user, is_admin
```

Логика:

- Если канал **приватный** (`ch["type"] == "private"`): разрешить только если текущий пользователь — участник канала (есть в `channel_members`) или админ (`is_admin()`). Иначе — `abort(403)` или редирект на дашборд с сообщением.
- Для публичных каналов дополнительной проверки не требуется.

После выборки канала и проверки на 404 добавьте:

```python
    if ch["type"] == "private":
        if is_admin():
            pass  # админ видит любой канал
        else:
            member = conn.execute(
                "SELECT 1 FROM channel_members WHERE channel_id = ? AND user_id = ?",
                (channel_id, current_user()["id"]),
            ).fetchone()
            if not member:
                conn.close()
                abort(403)
```

Учтите, что `conn` в этот момент открыт (вы только что делали запрос за каналом). Либо выполните проверку членства до `conn.close()`, либо получите соединение снова — как у вас организовано в текущем коде. Пример полной функции с одной связкой `get_conn()` и одним `conn.close()` в конце:

```python
def channel_view_view(channel_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    conn = get_conn()

    ch = conn.execute("""
        SELECT
            c.id,
            c.name,
            c.type,
            c.owner_id,
            c.created_at,
            u.username AS owner_name
        FROM channels c
        LEFT JOIN users u ON c.owner_id = u.id
        WHERE c.id = ?
    """, (channel_id,)).fetchone()

    if ch is None:
        conn.close()
        abort(404)

    # Приватный канал — только участники или админ
    if ch["type"] == "private":
        if not is_admin():
            member = conn.execute(
                "SELECT 1 FROM channel_members WHERE channel_id = ? AND user_id = ?",
                (channel_id, user["id"]),
            ).fetchone()
            if not member:
                conn.close()
                abort(403)

    # Роль текущего пользователя в канале (для шаблона: владелец, участник, только чтение, не в канале)
    membership = conn.execute(
        "SELECT role FROM channel_members WHERE channel_id = ? AND user_id = ?",
        (channel_id, user["id"]),
    ).fetchone()
    my_role = membership["role"] if membership else None

    conn.close()

    return render_template(
        "channel_view.html",
        channel=ch,
        my_role=my_role,
    )
```

В шаблон передаётся `my_role`: строка `owner` / `member` / `read_only` или `None`, если пользователь не в канале (для публичного канала тогда показываем кнопку «Вступить»).

---

## G7. Шаблоны: дашборд и страница канала

### Шаг G7.1. Дашборд: роль в «моих» каналах, кнопка «Вступить» для публичных

В `templates/dashboard.html`:

- В списке «Мои каналы» используйте `ch.my_role` для отображения роли (например, подпись «владелец», «участник», «только чтение») и оставьте ссылку на канал.
- В списке «Доступные публичные каналы» замените текст «(просмотр; вступление — в следующих частях)» на форму с кнопкой «Вступить», отправляющую POST на `url_for('channel_join', channel_id=ch.id)`.

Пример списка «Мои каналы» с ролью:

```html
    <h2>Мои каналы</h2>
    {% if not my_channels %}
      <p class="muted">Вы ещё не состоите ни в одном канале. Создайте канал или вступите в публичный.</p>
    {% else %}
      <ul>
        {% for ch in my_channels %}
          <li>
            <a href="{{ url_for('channel_view', channel_id=ch.id) }}">#{{ ch.name }}</a>
            <span class="muted">({{ ch.my_role }})</span>
            {% if ch.type == 'read_only' %}<span class="badge">read‑only</span>{% endif %}
          </li>
        {% endfor %}
      </ul>
    {% endif %}
```

Пример блока «Доступные публичные каналы» с кнопкой «Вступить» (POST):

```html
    <h2>Доступные публичные каналы</h2>
    {% if not public_joinable %}
      <p class="muted">Нет публичных каналов для вступления.</p>
    {% else %}
      <ul>
        {% for ch in public_joinable %}
          <li>
            #{{ ch.name }}
            <form method="post" action="{{ url_for('channel_join', channel_id=ch.id) }}" style="display: inline;">
              <button type="submit" class="btn btn-small">Вступить</button>
            </form>
          </li>
        {% endfor %}
      </ul>
    {% endif %}
```

Стиль кнопки `btn-small` можно добавить в общий CSS при необходимости или использовать существующий класс.

### Шаг G7.2. Страница канала: роль пользователя и кнопка «Покинуть канал»

В `templates/channel_view.html`:

- Выведите роль текущего пользователя в канале: если `my_role` передан — показать «Ваша роль: владелец» / «участник» / «только чтение»; если `my_role` нет (публичный канал, пользователь не вступил) — показать кнопку/форму «Вступить» (POST на `channel_join`).
- Для пользователей, которые **уже в канале** (т.е. `my_role` задана), показать форму «Покинуть канал» (POST на `url_for('channel_leave', channel_id=channel.id)`).

Пример фрагмента после заголовка канала:

```html
    <p class="muted">
      Тип: {{ channel.type }}<br />
      Владелец:
      {% if channel.owner_name %}
        {{ channel.owner_name }}
      {% else %}
        (не указан)
      {% endif %}
      <br />
      Создан: {{ channel.created_at }}
    </p>

    {% if my_role %}
      <p>Ваша роль в канале: <strong>{{ my_role }}</strong>.</p>
      <form method="post" action="{{ url_for('channel_leave', channel_id=channel.id) }}" style="display: inline;">
        <button type="submit" class="btn">Покинуть канал</button>
      </form>
    {% else %}
      {% if channel.type == 'public' %}
        <form method="post" action="{{ url_for('channel_join', channel_id=channel.id) }}" style="display: inline;">
          <button type="submit" class="btn btn-primary">Вступить в канал</button>
        </form>
      {% endif %}
    {% endif %}
```

Далее оставьте блок «Сообщения» и ссылку «К дашборду» как в Части F.

---

## Итог Части G (Slack)

После выполнения этой части в проекте есть:

- **`db.py`**: таблица `channel_members` в `init_db()` с полями `channel_id`, `user_id`, `role` (owner/member/read_only) и ограничением PRIMARY KEY (channel_id, user_id).
- **Создание канала**: в `channel_create_view()` после вставки в `channels` создатель добавляется в `channel_members` с ролью `owner`.
- **Дашборд**: «мои каналы» строятся по `channel_members` (с полем `my_role`); «доступные публичные каналы» — публичные каналы, в которых пользователя нет в `channel_members`, с кнопкой «Вступить».
- **Вступление**: `channel_join_view()` — только для публичных каналов, добавление в `channel_members` с ролью `member` или `read_only` в зависимости от типа канала; маршрут `POST /channels/<id>/join`.
- **Выход**: `channel_leave_view()` — удаление из `channel_members`, при выходе владельца — передача владения другому участнику или `owner_id = NULL`; маршрут `POST /channels/<id>/leave`.
- **Доступ к каналу**: в `channel_view_view()` для приватных каналов доступ только у участников и админа; в шаблон передаётся `my_role` для отображения роли и кнопок «Вступить» / «Покинуть канал».
- **Шаблоны**: `dashboard.html` обновлён (роль в «моих» каналах, форма «Вступить» для публичных); `channel_view.html` обновлён (роль, «Вступить» для не состоящих в публичном канале, «Покинуть канал» для участников).

Модель членства и сценарии «вступить / выйти» и «кто может открыть приватный канал» реализованы. В следующей части можно добавить управление участниками: список участников в канале, приглашение и удаление участников владельцем.
{% endraw %}
