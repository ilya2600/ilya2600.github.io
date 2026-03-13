{% raw %}
# Часть F: Заявки — первая предметная сущность

В Частях A–E вы подготовили каркас приложения и разнесли код по нескольким файлам:

- `app.py` — создание Flask‑приложения и маршруты;
- `db.py` — работа с базой данных (`init_db`, `get_conn` и т.п.);
- `auth_utils.py` — пользователи, логин, права (`current_user`, `is_admin`, `is_logged_in`, `ensure_master`);
- шаблоны в папке `templates/`.

В этой части вы добавите **первую предметную сущность** для трекера заявок — «заявка» (`ticket`):

- расширите схему БД таблицей `tickets`;
- создадите модуль `views_tickets.py` с логикой заявок;
- добавите маршруты в `app.py`;
- создадите шаблоны для списка заявок, создания и просмотра одной заявки.

Все шаги ниже даны **с привязкой к конкретным файлам** в текущем проекте.

---

## F1. Добавляем таблицу `tickets` в `db.py`

Сейчас файл `db.py` содержит `init_db()` только с таблицами `settings` и `users`. Нам нужно расширить её таблицей `tickets`.

### Шаг F1.1. Добавляем таблицу `tickets`

Откройте файл `db.py` и найдите функцию `def init_db()`. Сразу **после** блока `CREATE TABLE IF NOT EXISTS users (...)` вставьте ещё один `conn.execute`:

```python
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
```

Функция `init_db()` в итоге должна:

- создавать `settings`;
- создавать `users`;
- создавать `tickets`;
- вызывать `conn.commit()` и `conn.close()` (эти строки уже есть, их не трогаем).

---

## F2. Создаём модуль `views_tickets.py` с логикой заявок

Чтобы не перегружать `app.py`, всю логику заявок поместим в отдельный модуль.

### Шаг F2.1. Создаём файл

В той же папке, где лежит `app.py`, создайте новый файл:

- `views_tickets.py`

### Шаг F2.2. Импорты в `views_tickets.py`

В начало файла `views_tickets.py` вставьте:

```python
from flask import render_template, redirect, url_for, request, abort

from db import get_conn
from auth_utils import is_logged_in, current_user, is_admin
```

### Шаг F2.3. Функция `tickets_for_user`

Теперь **в этом же файле** `views_tickets.py` создайте функцию, которая будет возвращать список заявок, видимых текущему пользователю:

```python
def tickets_for_user():
    """Вернуть заявки, видимые текущему пользователю."""
    user = current_user()
    if user is None:
        return []

    conn = get_conn()

    if is_admin():
        # Администратор видит все заявки
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
        # Обычный пользователь видит заявки, где он автор или исполнитель
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

### Шаг F2.4. Функции прав: `can_view_ticket` и `can_edit_ticket`

В этом же файле `views_tickets.py`, **под** `tickets_for_user`, добавьте:

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

    if is_admin():
        return True, t

    if t["reporter_id"] == user["id"] or t["assignee_id"] == user["id"]:
        return True, t

    return False, None


def can_edit_ticket(ticket_id):
    """Можно ли редактировать эту заявку (менять статус, исполнителя и т.п.)."""
    user = current_user()
    if user is None:
        return False

    if is_admin():
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

### Шаг F2.5. View‑функции списка, создания и просмотра

Всё ещё в `views_tickets.py`, ниже хелперов прав, создайте view‑функции (без декораторов Flask):

```python
def ticket_list_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    tickets = tickets_for_user()
    return render_template("ticket_list.html", tickets=tickets)


def ticket_new_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    # В этом проекте создают заявки все залогиненные пользователи (включая админа)
    return render_template("ticket_new.html", error=None)


def ticket_create_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

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


def ticket_detail_view(ticket_id: int):
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

Теперь вся логика, связанная с заявками, находится в одном месте — `views_tickets.py`.

---

## F3. Подключаем маршруты заявок в `app.py`

Сейчас `app.py` содержит только маршруты для главной страницы, логина, админ‑части и т.п. Нам нужно добавить маршруты для заявок, которые будут вызывать функции из `views_tickets.py`.

### Шаг F3.1. Импортируем view‑функции

Откройте `app.py` и в блок импорта view‑модулей добавьте:

```python
from views_tickets import (
    ticket_list_view,
    ticket_new_view,
    ticket_create_view,
    ticket_detail_view,
)
```

Разместите этот импорт рядом с импортами `views_auth` и `views_admin`.

### Шаг F3.2. Добавляем маршруты

В `app.py`, **ниже** уже существующих маршрутов (например, после админ‑маршрутов), добавьте:

```python
@app.get("/tickets")
def ticket_list():
    return ticket_list_view()


@app.get("/tickets/new")
def ticket_new():
    return ticket_new_view()


@app.post("/tickets/new")
def ticket_create():
    return ticket_create_view()


@app.get("/tickets/<int:ticket_id>")
def ticket_detail(ticket_id):
    return ticket_detail_view(ticket_id)
```

Важно:

- **не** дублируйте эту логику в самом `app.py`;
- всё, что связано с БД и проверкой прав для заявок, уже есть в `views_tickets.py`.

---

## F4. Шаблон списка заявок `ticket_list.html`

### Шаг F4.1. Создаём файл шаблона

В папке `templates` создайте файл:

- `ticket_list.html`

### Шаг F4.2. Наполняем шаблон

Вставьте в `ticket_list.html`:

```html
{% extends "base.html" %}

{% block title %}Заявки{% endblock %}

{% block content %}
  <section class="card">
    <h1>Заявки</h1>

    <p>
      <a href="{{ url_for('ticket_new') }}" class="btn btn-primary">Новая заявка</a>
    </p>

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

---

## F5. Шаблон создания заявки `ticket_new.html`

### Шаг F5.1. Создаём файл шаблона

В папке `templates` создайте файл:

- `ticket_new.html`

### Шаг F5.2. Наполняем шаблон

Вставьте в `ticket_new.html`:

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

---

## F6. Шаблон просмотра заявки `ticket_detail.html`

### Шаг F6.1. Создаём файл шаблона

В папке `templates` создайте файл:

- `ticket_detail.html`

### Шаг F6.2. Наполняем шаблон

Вставьте в `ticket_detail.html`:

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

---

## F7. Ссылка на заявки в навигации

Если в вашем базовом шаблоне `base.html` есть горизонтальное меню, можно добавить туда ссылку на список заявок:

```html
<a href="{{ url_for('ticket_list') }}">Заявки</a>
```

Разместите её рядом с `Дашборд`.

---

## Итог Части F

После выполнения этой части у вас есть:

- расширенная схема БД с таблицей `tickets`;
- модуль `views_tickets.py` с функциями:
  - `tickets_for_user`,
  - `can_view_ticket`,
  - `can_edit_ticket`,
  - `ticket_list_view`,
  - `ticket_new_view`,
  - `ticket_create_view`,
  - `ticket_detail_view`;
- маршруты в `app.py`, которые используют эти функции и реализуют:
  - список заявок `/tickets`,
  - создание заявки `/tickets/new`,
  - просмотр одной заявки `/tickets/<id>`;
- три шаблона: `ticket_list.html`, `ticket_new.html`, `ticket_detail.html`.

Это первая полноценная часть предметной логики трекера заявок. В следующих частях вы сможете добавить:

- изменение статуса и назначение исполнителя;
- комментарии к заявкам и историю изменений;
- мягкое удаление заявок и комментариев по уже знакомому паттерну soft delete.
{% endraw %}

