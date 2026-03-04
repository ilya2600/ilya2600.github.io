{% raw %}
# Часть E: Заявки — первая предметная сущность

В Частях A–D вы подготовили универсальный каркас приложения:

- сервер Flask с базовым шаблоном и стилями;
- таблицу `users`, регистрацию, вход/выход, мастер‑аккаунт;
- флаг «регистрация открыта/закрыта»;
- минимальную админ‑страницу пользователей с архивированием и удалением.

В этой части мы наконец вводим **первую предметную сущность для трекера заявок** — «заявка» (`ticket`):

- добавим таблицу `tickets` в базу данных;
- напишем функции‑хелперы видимости и прав;
- сделаем список заявок, форму создания и страницу просмотра одной заявки.

Комментарии, смена статуса/исполнителя из формы, а также soft delete заявок мы пока откладываем на следующие части.

---

## E1. Таблица `tickets` в `init_db`

Сначала добавим таблицу для хранения заявок.

### 1. Поля и связи

Таблица `tickets` будет содержать:

- `id` — идентификатор заявки;
- `title` — короткий заголовок;
- `description` — текстовое описание;
- `reporter_id` — кто создал заявку (ссылка на `users.id`);
- `assignee_id` — кто отвечает за её выполнение (может быть `NULL`);
- `status` — статус заявки (`Open`, `In Progress`, `Blocked`, `Resolved`, `Closed`);
- `priority` — приоритет (`Low`, `Medium`, `High`);
- `created_at` — когда заявка была создана;
- `updated_at` — когда последний раз изменялась;
- `closed_at` — когда была закрыта (для открытых заявок `NULL`);
- `deleted_at` — метка мягкого удаления (позже пригодится для soft delete).

### 2. Добавляем `CREATE TABLE tickets` в `init_db`

Откройте `app.py`, найдите функцию `init_db()` и **после создания таблицы `users`** добавьте блок создания таблицы `tickets` и поменяйте `user` на `agent`:

```python
def init_db():
    ...
    # вместо 'users' -> 'agent'
    conn.execute("""
        CREATE TABLE IF NOT EXISTS users ...
        ...
        role TEXT NOT NULL CHECK (role IN ('agent', 'admin')),
        ...
    """)

    conn.execute("""
        CREATE TABLE IF NOT EXISTS tickets (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            description TEXT NOT NULL,
            reporter_id INTEGER,
            assignee_id INTEGER,
            status TEXT NOT NULL DEFAULT 'Open',
            priority TEXT NOT NULL DEFAULT 'Medium',
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            updated_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            closed_at TEXT,
            deleted_at TEXT,
            FOREIGN KEY (reporter_id) REFERENCES users(id),
            FOREIGN KEY (assignee_id) REFERENCES users(id)
        )
    """)

    conn.commit()
    conn.close()
```

> Главное — чтобы таблица `users` создавалась раньше, чем `tickets`, потому что в `tickets` на неё есть внешние ключи.

---

## E2. Хелперы видимости и прав для заявок

Дальше нам нужны функции, которые будут:

- определять, **какие заявки видит текущий пользователь**;
- проверять, можно ли **просматривать** и **редактировать** конкретную заявку.

Роли:

- **мастер** (`admin`) — видит все заявки;
- **агент** (`agent`) — видит только свои заявки:
  - где он автор (`reporter_id`),
  - или назначенный исполнитель (`assignee_id`).

Пока игнорируем поле `deleted_at` — soft delete заявок добавим позже.

### 1. Список заявок для текущего пользователя: `tickets_for_user`

Добавьте в `app.py` (рядом с другими хелперами для авторизации) функцию:

```python
VALID_STATUSES = ("Open", "In Progress", "Blocked", "Resolved", "Closed")


def tickets_for_user():
    """Вернуть заявки, видимые текущему пользователю."""
    user = current_user()
    if user is None:
        return []

    conn = get_conn()

    if is_master():
        # Мастер видит все заявки
        rows = conn.execute("""
            SELECT
                t.id,
                t.title,
                t.status,
                t.priority,
                t.created_at,
                t.reporter_id,
                t.assignee_id,
                ru.username AS reporter_username,
                ru.archived_at AS reporter_archived,
                au.username AS assignee_username,
                au.archived_at AS assignee_archived
            FROM tickets t
            LEFT JOIN users ru ON t.reporter_id = ru.id
            LEFT JOIN users au ON t.assignee_id = au.id
            ORDER BY t.created_at DESC
        """).fetchall()
    else:
        # Агент видит только свои заявки: как автор или как исполнитель
        rows = conn.execute("""
            SELECT
                t.id,
                t.title,
                t.status,
                t.priority,
                t.created_at,
                t.reporter_id,
                t.assignee_id,
                ru.username AS reporter_username,
                ru.archived_at AS reporter_archived,
                au.username AS assignee_username,
                au.archived_at AS assignee_archived
            FROM tickets t
            LEFT JOIN users ru ON t.reporter_id = ru.id
            LEFT JOIN users au ON t.assignee_id = au.id
            WHERE t.reporter_id = ? OR t.assignee_id = ?
            ORDER BY t.created_at DESC
        """, (user["id"], user["id"])).fetchall()

    conn.close()
    return rows
```

### 2. Проверка, можно ли видеть заявку: `can_view_ticket`

Функция должна возвращать:

- `True`/`False` — есть ли доступ;
- (опционально) саму строку заявки, если она найдена.

Добавьте:

```python
def can_view_ticket(ticket_id):
    """Можно ли просматривать эту заявку текущему пользователю."""
    user = current_user()
    if user is None:
        return False, None

    conn = get_conn()
    t = conn.execute(
        "SELECT id, reporter_id, assignee_id FROM tickets WHERE id = ?",
        (ticket_id,),
    ).fetchone()
    conn.close()

    if t is None:
        return False, None

    if is_master():
        return True, t

    if is_agent() and (t["reporter_id"] == user["id"] or t["assignee_id"] == user["id"]):
        return True, t

    return False, None
```

### 3. Проверка, можно ли редактировать заявку: `can_edit_ticket`

Редактировать заявку (в следующих частях) смогут:

- мастер — любую;
- агент — только те, где он автор или исполнитель.

Добавьте:

```python
def can_edit_ticket(ticket_id):
    """Можно ли редактировать эту заявку (менять статус, исполнителя и т.п.)."""
    user = current_user()
    if user is None:
        return False

    if is_master():
        return True

    conn = get_conn()
    t = conn.execute(
        "SELECT reporter_id, assignee_id FROM tickets WHERE id = ?",
        (ticket_id,),
    ).fetchone()
    conn.close()

    if t is None:
        return False

    return t["reporter_id"] == user["id"] or t["assignee_id"] == user["id"]
```

Эти функции будут использоваться в маршрутах списка, просмотра и (позже) обновления заявки.

---

## E3. Список заявок `/tickets`

Сделаем страницу, на которой пользователь увидит все заявки, к которым у него есть доступ.

### 1. Маршрут `GET /tickets`

Добавьте в `app.py`:

```python
@app.get("/tickets")
def ticket_list():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    tickets = tickets_for_user()
    return render_template("ticket_list.html", tickets=tickets)
```

### 2. Шаблон `ticket_list.html`

Создайте файл `templates/ticket_list.html` со списком заявок в таблице:

```html
{% extends "base.html" %}

{% block title %}Заявки{% endblock %}

{% block content %}
  <section class="card">
    <h1>Заявки</h1>

    {% if is_master() %}
      <p class="muted">
        Мастер видит все заявки. Позже здесь можно будет добавить фильтры и soft delete.
      </p>
    {% endif %}

    {% if is_master() or is_agent() %}
      <p>
        <a href="{{ url_for('ticket_new') }}" class="btn btn-primary">Новая заявка</a>
      </p>
    {% endif %}

    {% if not tickets %}
      <p class="muted">Пока нет ни одной заявки.</p>
    {% else %}
      <table class="table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Заголовок</th>
            <th>Статус</th>
            <th>Приоритет</th>
            <th>Автор</th>
            <th>Исполнитель</th>
            <th>Создана</th>
          </tr>
        </thead>
        <tbody>
          {% for t in tickets %}
            <tr>
              <td>
                <a href="{{ url_for('ticket_detail', ticket_id=t.id) }}">{{ t.id }}</a>
              </td>
              <td>
                <a href="{{ url_for('ticket_detail', ticket_id=t.id) }}">{{ t.title }}</a>
              </td>
              <td>{{ t.status }}</td>
              <td>{{ t.priority }}</td>
              <td>
                {% if t.reporter_id and not t.reporter_archived %}
                  {{ t.reporter_username }}
                {% elif t.reporter_id %}
                  Удалённый пользователь
                {% else %}
                  —
                {% endif %}
              </td>
              <td>
                {% if t.assignee_id %}
                  {% if not t.assignee_archived %}
                    {{ t.assignee_username }}
                  {% else %}
                    Удалённый пользователь
                  {% endif %}
                {% else %}
                  —
                {% endif %}
              </td>
              <td>{{ t.created_at[:10] if t.created_at else '' }}</td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    {% endif %}
  </section>
{% endblock %}
```

### 3. Ссылка на список заявок в навигации

В `templates/base.html` (в блоке навигации для вошедшего пользователя) добавьте ссылку на `/tickets`:

```html
<nav>
  <a href="{{ url_for('home') }}">Главная</a>
  {% if current_user() %}
    <a href="{{ url_for('dashboard') }}">Дашборд</a>
    <a href="{{ url_for('ticket_list') }}">Заявки</a>
    <!-- остальные ссылки для мастера/админа при необходимости -->
  {% else %}
    <a href="{{ url_for('login_form') }}">Вход</a>
    {% if registration_open %}
      <a href="{{ url_for('register') }}">Регистрация</a>
    {% endif %}
  {% endif %}
</nav>
```

---

## E4. Создание новой заявки

Теперь добавим форму и маршруты для создания заявки.

### 1. Маршруты `GET /tickets/new` и `POST /tickets/new`

В `app.py` добавьте два маршрута:

```python
@app.get("/tickets/new")
def ticket_new():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    # Создавать заявки могут агенты и мастер
    if not (is_agent() or is_master()):
        abort(403)

    return render_template("ticket_new.html", error=None)


@app.post("/tickets/new")
def ticket_create():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    if not (is_agent() or is_master()):
        abort(403)

    title = (request.form.get("title") or "").strip()
    description = (request.form.get("description") or "").strip()
    priority = request.form.get("priority") or "Medium"

    if not title:
        return render_template("ticket_new.html", error="Введите заголовок заявки.")

    if priority not in ("Low", "Medium", "High"):
        priority = "Medium"

    user = current_user()

    conn = get_conn()
    conn.execute(
        "INSERT INTO tickets (title, description, reporter_id, status, priority) "
        "VALUES (?, ?, ?, 'Open', ?)",
        (title, description, user["id"], priority),
    )
    conn.commit()
    conn.close()

    return redirect(url_for("ticket_list"))
```

### 2. Шаблон `ticket_new.html`

Создайте файл `templates/ticket_new.html`:

```html
{% extends "base.html" %}

{% block title %}Новая заявка{% endblock %}

{% block content %}
  <section class="card">
    <h1>Новая заявка</h1>

    {% if error %}
      <p class="bad">{{ error }}</p>
    {% endif %}

    <form method="post" action="{{ url_for('ticket_create') }}" class="form">
      <div class="form-group">
        <label for="title">Заголовок</label>
        <input type="text" id="title" name="title" required />
      </div>

      <div class="form-group">
        <label for="description">Описание</label>
        <textarea id="description" name="description" rows="5"></textarea>
      </div>

      <div class="form-group">
        <label for="priority">Приоритет</label>
        <select id="priority" name="priority">
          <option value="Low">Low</option>
          <option value="Medium" selected>Medium</option>
          <option value="High">High</option>
        </select>
      </div>

      <button type="submit" class="btn btn-primary">Создать заявку</button>
    </form>

    <p style="margin-top: 16px;">
      <a href="{{ url_for('ticket_list') }}">К списку заявок</a>
    </p>
  </section>
{% endblock %}
```

После этого по ссылке «Новая заявка» на странице списка можно будет открыть форму, заполнить заголовок, описание и приоритет, а затем вернуться к списку уже с новой строкой.

---

## E5. Просмотр одной заявки

Осталось сделать страницу, на которой можно подробно посмотреть информацию по одной заявке.

### 1. Маршрут `GET /tickets/<int:ticket_id>`

Добавьте в `app.py`:

```python
@app.get("/tickets/<int:ticket_id>")
def ticket_detail(ticket_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    can_view, _ = can_view_ticket(ticket_id)
    if not can_view:
        # Если заявки нет или нет прав — возвращаем 404,
        # чтобы не раскрывать информацию о существовании заявки.
        abort(404)

    conn = get_conn()
    t = conn.execute("""
        SELECT
            t.id,
            t.title,
            t.description,
            t.status,
            t.priority,
            t.created_at,
            t.updated_at,
            t.closed_at,
            t.reporter_id,
            t.assignee_id,
            ru.username AS reporter_username,
            ru.archived_at AS reporter_archived,
            au.username AS assignee_username,
            au.archived_at AS assignee_archived
        FROM tickets t
        LEFT JOIN users ru ON t.reporter_id = ru.id
        LEFT JOIN users au ON t.assignee_id = au.id
        WHERE t.id = ?
    """, (ticket_id,)).fetchone()
    conn.close()

    if t is None:
        abort(404)

    return render_template("ticket_detail.html", ticket=t, can_edit=can_edit_ticket(ticket_id))
```

### 2. Шаблон `ticket_detail.html`

Создайте файл `templates/ticket_detail.html` с простой «карточкой» заявки:

```html
{% extends "base.html" %}

{% block title %}Заявка #{{ ticket.id }}{% endblock %}

{% block content %}
  <section class="card">
    <h1>Заявка #{{ ticket.id }}</h1>

    <p><strong>Заголовок:</strong> {{ ticket.title }}</p>

    <p>
      <strong>Статус:</strong> {{ ticket.status }}<br />
      <strong>Приоритет:</strong> {{ ticket.priority }}
    </p>

    <p>
      <strong>Автор:</strong>
      {% if ticket.reporter_id and not ticket.reporter_archived %}
        {{ ticket.reporter_username }}
      {% elif ticket.reporter_id %}
        Удалённый пользователь
      {% else %}
        —
      {% endif %}
      <br />
      <strong>Исполнитель:</strong>
      {% if ticket.assignee_id %}
        {% if not ticket.assignee_archived %}
          {{ ticket.assignee_username }}
        {% else %}
          Удалённый пользователь
        {% endif %}
      {% else %}
        —
      {% endif %}
    </p>

    <p>
      <strong>Создана:</strong> {{ ticket.created_at }}<br />
      {% if ticket.updated_at %}
        <strong>Обновлена:</strong> {{ ticket.updated_at }}
      {% endif %}
      {% if ticket.closed_at %}
        <br /><strong>Закрыта:</strong> {{ ticket.closed_at }}
      {% endif %}
    </p>

    <h2>Описание</h2>
    <pre>{{ ticket.description }}</pre>

    <p style="margin-top: 16px;">
      <a href="{{ url_for('ticket_list') }}">К списку заявок</a>
    </p>
  </section>
{% endblock %}
```

На этом шаге мы **не добавляем** форм для изменения статуса/исполнителя и **не выводим комментарии** — только чтение данных по заявке. Эти функции появятся в следующих частях.

---

## Итог Части E

После выполнения этой части у вас есть:

- таблица `tickets` с основными полями для заявок;
- хелперы `tickets_for_user`, `can_view_ticket`, `can_edit_ticket` с учётом ролей мастера и агента;
- список заявок `/tickets` с таблицей;
- форма создания заявки `/tickets/new`;
- страница просмотра одной заявки `/tickets/<id>`.

Это первая часть предметной логики трекера заявок. В следующих частях вы сможете добавить:

- изменение статуса и назначение исполнителя;
- комментарии к заявкам и историю изменений;
- мягкое удаление заявок и комментариев по уже знакомому паттерну soft delete.
{% endraw %}

