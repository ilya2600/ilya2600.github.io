{% raw %}
# Часть G: Проекты и привязка заявок к проектам

В Части F вы добавили **первые заявки (`tickets`)**:

- таблица `tickets` в `db.py`,
- модуль `views_tickets.py` с логикой,
- маршруты `/tickets`, `/tickets/new`, `/tickets/<id>`,
- шаблоны `ticket_list.html`, `ticket_new.html`, `ticket_detail.html`.

Пока все заявки глобальные — они не принадлежат проектам.  
В этой части мы добавим **проекты** и сделаем так, чтобы новые заявки создавались **внутри конкретного проекта**.

> Важно: здесь мы **не вводим ещё роли в проектах и архивирование** — только сами проекты и простую привязку заявок к проектам.

---

## G0. Подготовка и важное замечание про базу

В этой части мы **меняем схему базы данных**:

- добавляем новую таблицу `projects`,
- добавляем в таблицу `tickets` новое поле `project_id`.

SQLite не очень удобен для сложных миграций, поэтому для учебного проекта проще всего:

1. Скопировать (при необходимости) файл `database.db` в какое‑нибудь безопасное место, если вы не хотите терять данные.
2. **Удалить** текущий `database.db` из корня проекта.
3. После выполнения шагов по изменению `init_db()` просто заново запустить приложение — оно создаст чистую базу с новой схемой.

Если вы оставите старую базу, новые `CREATE TABLE` и поле `project_id` могут не появиться как вы ожидаете.

---

## G1. Таблица `projects` и поле `project_id` в `tickets`

Откройте файл `db.py`.

### G1.1. Добавляем таблицу `projects`

Найдите функцию:

```python
def init_db():
    ...
```

Внутри неё уже создаются таблицы `settings` и `users`, а также `tickets`.  
Между созданием `users` и `tickets` добавьте создание `projects`:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS projects (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            owner_id INTEGER NOT NULL,
            title TEXT NOT NULL,
            description TEXT,
            is_archived INTEGER NOT NULL DEFAULT 0,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            FOREIGN KEY (owner_id) REFERENCES users(id)
        )
    """)
```

Кратко по полям:

- `owner_id` — владелец проекта (создатель),
- `title` — название,
- `description` — описание (может быть пустым),
- `is_archived` — пока всегда `0`, архивирование появится в Части J,
- `created_at` — время создания.

### G1.2. Добавляем `project_id` в `tickets`

В той же функции `init_db()` найдите блок `CREATE TABLE IF NOT EXISTS tickets (...)` и дополните его:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS tickets (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            description TEXT NOT NULL,
            reporter_id INTEGER,
            assignee_id INTEGER,
            project_id INTEGER,  -- новое поле
            status TEXT NOT NULL DEFAULT 'Open',
            priority TEXT NOT NULL DEFAULT 'Medium',
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            updated_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            closed_at TEXT,
            deleted_at TEXT,
            FOREIGN KEY (reporter_id) REFERENCES users(id),
            FOREIGN KEY (assignee_id) REFERENCES users(id),
            FOREIGN KEY (project_id) REFERENCES projects(id)
        )
    """)
```

> Следите за запятыми: после `assignee_id INTEGER` нужна запятая, и в конце списка полей после последнего `FOREIGN KEY` запятая не нужна.

После правок сохраните файл `db.py`.  
Позже мы перезапустим приложение, чтобы создать таблицы с новой схемой.

---

## G2. Новый модуль `views_projects.py`

Сделаем отдельный модуль с логикой проектов, по аналогии с `views_auth.py` и `views_tickets.py`.

### G2.1. Создаём файл

Создайте в корне проекта (рядом с `app.py`) новый файл:

- `views_projects.py`

### G2.2. Импорты и простые хелперы

Откройте `views_projects.py` и вставьте базовый код:

```python
from flask import render_template, redirect, url_for, request, abort

from db import get_conn
from auth_utils import is_logged_in, current_user


def get_project(project_id):
    """Вернуть проект по id или None, если не найден."""
    conn = get_conn()
    project = conn.execute(
        "SELECT * FROM projects WHERE id = ?",
        (project_id,),
    ).fetchone()
    conn.close()
    return project


def user_owns_project(project_id):
    """Проверить, что текущий пользователь — владелец проекта."""
    user = current_user()
    if user is None:
        return False

    project = get_project(project_id)
    if project is None:
        return False

    return project["owner_id"] == user["id"]
```

На этом шаге:

- **нет ещё участников проекта и ролей** — только владелец,
- все проверки завязаны на `current_user()` и поле `owner_id`.

### G2.3. Список проектов владельца

Добавим функцию, которая показывает все проекты текущего пользователя:

```python
def projects_list_view():
    """Список проектов текущего пользователя (как владелец)."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    if user is None:
        return redirect(url_for("login_form"))

    conn = get_conn()
    projects = conn.execute(
        """
        SELECT id, title, description, is_archived, created_at
        FROM projects
        WHERE owner_id = ?
        ORDER BY created_at DESC
        """,
        (user["id"],),
    ).fetchall()
    conn.close()

    return render_template("project_list.html", projects=projects)
```

### G2.4. Создание нового проекта

В том же файле добавьте две функции: для формы и для приёма POST:

```python
def project_new_view():
    """Показать форму создания проекта."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    return render_template("project_new.html", error=None)


def project_create_view():
    """Обработать создание проекта."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    if user is None:
        return redirect(url_for("login_form"))

    title = (request.form.get("title") or "").strip()
    description = (request.form.get("description") or "").strip()

    if not title:
        return render_template(
            "project_new.html",
            error="Введите название проекта.",
        )

    conn = get_conn()
    conn.execute(
        "INSERT INTO projects (owner_id, title, description) VALUES (?, ?, ?)",
        (user["id"], title, description),
    )
    conn.commit()
    conn.close()

    return redirect(url_for("projects_list"))
```

Здесь:

- при ошибке валидации (пустой заголовок) мы просто снова показываем форму,
- при успехе — перенаправляем на список проектов.

### G2.5. Просмотр конкретного проекта

Наконец, добавим представление для страницы одного проекта:

```python
def project_detail_view(project_id):
    """Страница одного проекта с его заявками."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    if user is None:
        return redirect(url_for("login_form"))

    project = get_project(project_id)
    if project is None:
        abort(404)

    # На этом шаге даём доступ только владельцу проекта
    if project["owner_id"] != user["id"]:
        abort(403)

    conn = get_conn()
    tickets = conn.execute(
        """
        SELECT
            t.id,
            t.title,
            t.status,
            t.priority,
            t.created_at,
            ru.username AS reporter_username
        FROM tickets t
        LEFT JOIN users ru ON t.reporter_id = ru.id
        WHERE t.project_id = ?
        ORDER BY t.created_at DESC
        """,
        (project_id,),
    ).fetchall()
    conn.close()

    return render_template(
        "project_detail.html",
        project=project,
        tickets=tickets,
    )
```

Сейчас:

- видит проект только владелец,
- внизу страницы мы покажем список заявок этого проекта (шаблон сделаем далее),
- в будущем сюда добавятся разделы «Участники», «История» и т.п.

---

## G3. Маршруты проектов в `app.py`

Теперь подключим новый модуль к приложению.

### G3.1. Импортируем `views_projects`

Откройте `app.py` и в начало файла, рядом с импортами `views_auth`, `views_tickets`, `views_admin`, добавьте:

```python
from views_projects import (
    projects_list_view,
    project_new_view,
    project_create_view,
    project_detail_view,
)
```

### G3.2. Добавляем маршруты

Внизу файла, рядом с другими маршрутами, добавьте новые:

```python
@app.get("/projects")
def projects_list():
    return projects_list_view()


@app.get("/projects/new")
def project_new():
    return project_new_view()


@app.post("/projects/new")
def project_create():
    return project_create_view()


@app.get("/projects/<int:project_id>")
def project_detail(project_id):
    return project_detail_view(project_id)
```

Мы не убираем маршрут `/dashboard` — он по‑прежнему показывает простую страницу.  
В Части H–J вы можете дополнительно «обогатить» дашборд, но для этой части достаточно отдельных страниц `/projects...`.

---

## G4. Привязываем создание заявок к проектам

Сейчас заявки создаются через маршруты:

- `GET /tickets`,
- `GET /tickets/new`,
- `POST /tickets/new`,
- `GET /tickets/<id>`.

Наша цель — чтобы новые заявки создавались **из контекста проекта**. Для этого удобнее всего сделать проект‑зависимые маршруты.

> Это **изменение URL по сравнению с Частью F**.  
> Старые адреса `/tickets/new` и `/tickets/<id>` заменяются на маршруты вида `/projects/<project_id>/tickets/...`.  
> Это нормально — мы двигаемся по шагам к более сложной структуре.

### G4.1. Обновляем маршруты в `app.py`

В `app.py` найдите существующие маршруты:

```python
@app.get("/tickets")
def ticket_list():
    ...

@app.get("/tickets/new")
def ticket_new():
    ...

@app.post("/tickets/new")
def ticket_create():
    ...

@app.get("/tickets/<int:ticket_id>")
def ticket_detail(ticket_id):
    ...
```

Замените их на варианты с `project_id`:

```python
@app.get("/projects/<int:project_id>/tickets")
def ticket_list(project_id):
    return ticket_list_view(project_id)


@app.get("/projects/<int:project_id>/tickets/new")
def ticket_new(project_id):
    return ticket_new_view(project_id)


@app.post("/projects/<int:project_id>/tickets/new")
def ticket_create(project_id):
    return ticket_create_view(project_id)


@app.get("/projects/<int:project_id>/tickets/<int:ticket_id>")
def ticket_detail(project_id, ticket_id):
    return ticket_detail_view(project_id, ticket_id)
```

Обратите внимание:

- мы передаём `project_id` внутрь view‑функций,
- позже мы обновим сами функции в `views_tickets.py`, чтобы они учитывали проект.

### G4.2. Обновляем `views_tickets.py` под проект

Откройте файл `views_tickets.py`.

#### G4.2.1. Список заявок

Сейчас функция `tickets_for_user()` не знает ничего о проектах.  
Для упрощения, в Части G мы **внутри проекта** показываем **все заявки этого проекта**, не разделяя их по пользователям. Глобальные фильтры по пользователю останутся в будущем шаге.

Обновите `tickets_for_user` так, чтобы она принимала `project_id`:

```python
def tickets_for_project(project_id):
    """Вернуть все заявки этого проекта (простая версия)."""
    conn = get_conn()
    rows = conn.execute(
        """
        SELECT
            t.id,
            t.title,
            t.status,
            t.priority,
            t.created_at,
            t.reporter_id,
            t.assignee_id,
            ru.username AS reporter_username,
            au.username AS assignee_username
        FROM tickets t
        LEFT JOIN users ru ON t.reporter_id = ru.id
        LEFT JOIN users au ON t.assignee_id = au.id
        WHERE t.project_id = ?
        ORDER BY t.created_at DESC
        """,
        (project_id,),
    ).fetchall()
    conn.close()
    return rows
```

И обновите `ticket_list_view`, чтобы использовать её:

```python
def ticket_list_view(project_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    tickets = tickets_for_project(project_id)
    return render_template("ticket_list.html", tickets=tickets, project_id=project_id)
```

> Старую функцию `tickets_for_user()` можно временно удалить или оставить неиспользуемой — в этой части мы больше строим логику вокруг проектов, а не вокруг «моих заявок».

#### G4.2.2. Создание заявки

Обновите `ticket_new_view`, чтобы принимать `project_id` и передавать его в шаблон:

```python
def ticket_new_view(project_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    return render_template("ticket_new.html", error=None, project_id=project_id)
```

Обновите `ticket_create_view`, чтобы:

- принимать `project_id`,
- записывать его в таблицу `tickets`.

```python
def ticket_create_view(project_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    title = (request.form.get("title") or "").strip()
    description = (request.form.get("description") or "").strip()
    priority = request.form.get("priority") or "Medium"

    if not title:
        return render_template(
            "ticket_new.html",
            error="Введите заголовок заявки.",
            project_id=project_id,
        )

    if priority not in ("Low", "Medium", "High"):
        priority = "Medium"

    user = current_user()

    conn = get_conn()
    conn.execute(
        "INSERT INTO tickets (title, description, reporter_id, project_id, status, priority) "
        "VALUES (?, ?, ?, ?, 'Open', ?)",
        (title, description, user["id"], project_id, priority),
    )
    conn.commit()
    conn.close()

    return redirect(url_for("ticket_list", project_id=project_id))
```

#### G4.2.3. Детали заявки

Обновите `ticket_detail_view`, чтобы принимать `project_id` и проверять, что заявка относится к этому проекту:

```python
def ticket_detail_view(project_id: int, ticket_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    can_view, _ = can_view_ticket(ticket_id)
    if not can_view:
        abort(404)

    conn = get_conn()
    t = conn.execute(
        """
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
            t.project_id,
            ru.username AS reporter_username,
            au.username AS assignee_username
        FROM tickets t
        LEFT JOIN users ru ON t.reporter_id = ru.id
        LEFT JOIN users au ON t.assignee_id = au.id
        WHERE t.id = ?
        """,
        (ticket_id,),
    ).fetchone()
    conn.close()

    if t is None or t["project_id"] != project_id:
        abort(404)

    return render_template(
        "ticket_detail.html",
        ticket=t,
        can_edit=can_edit_ticket(ticket_id),
        project_id=project_id,
    )
```

> Проверка `t["project_id"] != project_id` защищает от ситуаций, когда кто‑то вручную меняет URL с чужим `project_id`.

---

## G5. Новые шаблоны проектов

### G5.1. `project_list.html`

В папке `templates` создайте файл `project_list.html`:

```html
{% extends "base.html" %}

{% block title %}Проекты · Учебное веб‑приложение{% endblock %}

{% block content %}
  <section class="card">
    <h2>Мои проекты</h2>

    <p>
      <a href="{{ url_for('project_new') }}" class="btn btn-primary">Новый проект</a>
    </p>

    {% if not projects %}
      <p class="muted">У вас пока нет проектов.</p>
    {% else %}
      <table class="table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Название</th>
            <th>Описание</th>
            <th>Создан</th>
          </tr>
        </thead>
        <tbody>
          {% for p in projects %}
            <tr>
              <td>
                <a href="{{ url_for('project_detail', project_id=p.id) }}">{{ p.id }}</a>
              </td>
              <td>
                <a href="{{ url_for('project_detail', project_id=p.id) }}">{{ p.title }}</a>
              </td>
              <td>{{ p.description or '' }}</td>
              <td>{{ p.created_at[:10] if p.created_at else '' }}</td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    {% endif %}
  </section>
{% endblock %}
```

### G5.2. `project_new.html`

Создайте файл `project_new.html`:

```html
{% extends "base.html" %}

{% block title %}Новый проект · Учебное веб‑приложение{% endblock %}

{% block content %}
  <section class="card">
    <h2>Новый проект</h2>

    {% if error %}
      <p class="bad">{{ error }}</p>
    {% endif %}

    <form method="post" action="{{ url_for('project_create') }}" class="form">
      <div class="form-group">
        <label for="title">Название</label>
        <input type="text" id="title" name="title" required />
      </div>

      <div class="form-group">
        <label for="description">Описание (необязательно)</label>
        <textarea id="description" name="description" rows="4"></textarea>
      </div>

      <button type="submit" class="btn btn-primary">Создать проект</button>
    </form>

    <p style="margin-top: 16px;">
      <a href="{{ url_for('projects_list') }}">К списку проектов</a>
    </p>
  </section>
{% endblock %}
```

### G5.3. `project_detail.html`

Создайте файл `project_detail.html`:

```html
{% extends "base.html" %}

{% block title %}Проект #{{ project.id }} · Учебное веб‑приложение{% endblock %}

{% block content %}
  <section class="card">
    <h2>Проект #{{ project.id }} — {{ project.title }}</h2>

    {% if project.description %}
      <p>{{ project.description }}</p>
    {% endif %}

    <p>
      <strong>Создан:</strong> {{ project.created_at }}
    </p>

    <p style="margin-top: 12px;">
      <a href="{{ url_for('projects_list') }}">К списку проектов</a>
    </p>
  </section>

  <section class="card" style="margin-top: 16px;">
    <h3>Заявки этого проекта</h3>

    <p>
      <a href="{{ url_for('ticket_new', project_id=project.id) }}" class="btn btn-primary">
        Новая заявка
      </a>
    </p>

    {% if not tickets %}
      <p class="muted">В этом проекте пока нет заявок.</p>
    {% else %}
      <table class="table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Заголовок</th>
            <th>Статус</th>
            <th>Приоритет</th>
            <th>Автор</th>
            <th>Создана</th>
          </tr>
        </thead>
        <tbody>
          {% for t in tickets %}
            <tr>
              <td>
                <a href="{{ url_for('ticket_detail', project_id=project.id, ticket_id=t.id) }}">
                  {{ t.id }}
                </a>
              </td>
              <td>
                <a href="{{ url_for('ticket_detail', project_id=project.id, ticket_id=t.id) }}">
                  {{ t.title }}
                </a>
              </td>
              <td>{{ t.status }}</td>
              <td>{{ t.priority }}</td>
              <td>{{ t.reporter_username or '—' }}</td>
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

## G6. Обновляем шаблоны заявок под проекты

### G6.1. `ticket_list.html`

Откройте `templates/ticket_list.html`. Сейчас он ничего не знает о проектах и не принимает `project_id`.

Обновите его так, чтобы он:

- показывал ссылку обратно в проект,
- строил ссылки на детали заявки с учётом `project_id`.

Пример:

```html
{% extends "base.html" %}

{% block title %}Заявки проекта{% endblock %}

{% block content %}
  <section class="card">
    <h2>Заявки проекта</h2>

    <p>
      <a href="{{ url_for('project_detail', project_id=project_id) }}">Назад к проекту</a>
    </p>

    <p>
      <a href="{{ url_for('ticket_new', project_id=project_id) }}" class="btn btn-primary">
        Новая заявка
      </a>
    </p>

    {% if not tickets %}
      <p class="muted">В этом проекте пока нет заявок.</p>
    {% else %}
      <table class="table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Заголовок</th>
            <th>Статус</th>
            <th>Приоритет</th>
            <th>Создана</th>
          </tr>
        </thead>
        <tbody>
          {% for t in tickets %}
            <tr>
              <td>
                <a href="{{ url_for('ticket_detail', project_id=project_id, ticket_id=t.id) }}">
                  {{ t.id }}
                </a>
              </td>
              <td>
                <a href="{{ url_for('ticket_detail', project_id=project_id, ticket_id=t.id) }}">
                  {{ t.title }}
                </a>
              </td>
              <td>{{ t.status }}</td>
              <td>{{ t.priority }}</td>
              <td>{{ t.created_at[:10] if t.created_at else '' }}</td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    {% endif %}
  </section>
{% endblock %}
```

### G6.2. `ticket_new.html`

Откройте `templates/ticket_new.html` и добавьте поддержку `project_id`:

- ссылку «назад в проект»,
- корректный `action` формы.

Пример:

```html
{% extends "base.html" %}

{% block title %}Новая заявка{% endblock %}

{% block content %}
  <section class="card">
    <h2>Новая заявка</h2>

    <p>
      <a href="{{ url_for('project_detail', project_id=project_id) }}">Назад к проекту</a>
    </p>

    {% if error %}
      <p class="bad">{{ error }}</p>
    {% endif %}

    <form method="post" action="{{ url_for('ticket_create', project_id=project_id) }}" class="form">
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
  </section>
{% endblock %}
```

### G6.3. `ticket_detail.html`

Откройте `templates/ticket_detail.html` и добавьте:

- ссылку назад в проект,
- использование `project_id` в ссылке.

Пример:

```html
{% extends "base.html" %}

{% block title %}Заявка #{{ ticket.id }}{% endblock %}

{% block content %}
  <section class="card">
    <h2>Заявка #{{ ticket.id }}</h2>

    <p>
      <a href="{{ url_for('project_detail', project_id=project_id) }}">Назад к проекту</a>
    </p>

    <p><strong>Заголовок:</strong> {{ ticket.title }}</p>
    <p><strong>Статус:</strong> {{ ticket.status }}</p>
    <p><strong>Приоритет:</strong> {{ ticket.priority }}</p>

    <p>
      <strong>Автор:</strong> {{ ticket.reporter_username or '—' }}<br />
      <strong>Создана:</strong> {{ ticket.created_at }}
    </p>

    <h3>Описание</h3>
    <pre>{{ ticket.description }}</pre>
  </section>
{% endblock %}
```

---

## G7. Ссылка на проекты в шапке

Откройте `templates/base.html`.

Сейчас в навигации для авторизованного пользователя есть ссылки:

- «Главная»,
- «Дашборд»,
- «Заявки» (на глобальный список `/tickets`, который мы только что перенастроили).

Мы хотим, чтобы:

- появилась явная ссылка на список проектов,
- ссылка «Заявки» вела в контекст проектов или была убрана.

Простой вариант — заменить ссылку «Заявки» на «Проекты»:

```html
{% if current_user() %}
  <a href="{{ url_for('dashboard') }}">Дашборд</a>
  <a href="{{ url_for('projects_list') }}">Проекты</a>
  ...
{% endif %}
```

Так пользователь всегда может попасть к своим проектам и уже из проекта — к заявкам.

---

## G8. Проверка Части G (Visible / testable)

1. **Пересоздайте базу.**
   - Удалите `database.db` в корне проекта.
   - Запустите приложение: `python app.py`.
   - Убедитесь, что ошибок при создании схемы нет.

2. **Создайте проект.**
   - Залогиньтесь под мастером (`master`/`master`) или другим пользователем.
   - В шапке перейдите в «Проекты».
   - Нажмите «Новый проект», введите название и (опционально) описание.
   - После сохранения новый проект должен появиться в списке.

3. **Откройте проект.**
   - Кликните по проекту в списке.
   - На странице проекта (`/projects/<id>`) увидите:
     - заголовок и описание,
     - блок «Заявки этого проекта» с кнопкой «Новая заявка» и пустой таблицей.

4. **Создайте заявку внутри проекта.**
   - На странице проекта нажмите «Новая заявка».
   - Заполните форму и сохраните.
   - После сохранения вы должны вернуться к списку заявок проекта (или к странице проекта), где новая заявка видна в таблице.

5. **Откройте заявку.**
   - Кликните по заявке в списке.
   - На странице заявки:
     - есть ссылка «Назад к проекту»,
     - видно заголовок, статус, приоритет, автора и описание.

6. **Попробуйте открыть проект другого пользователя.**
   - Создайте ещё одного пользователя (через `/admin/users`).
   - Выйдите и залогиньтесь под другим пользователем.
   - Попробуйте ввести в адресе `/projects/<id>` для проекта, который принадлежит первому пользователю.
   - Ожидаемый результат — код 403 или страница «Доступ запрещён» (в зависимости от того, как вы настроите обработку `abort(403)`).

---

## Замечания и потенциальные конфликты

- **Изменение URL для заявок.**  
  В Части F заявки жили по адресам `/tickets`, `/tickets/new`, `/tickets/<id>`.  
  В Части G мы переносим их в пространство `/projects/<project_id>/tickets/...`.  
  Если у вас были сохранённые ссылки или автотесты на старые URL, их нужно будет обновить.

- **Сброс базы данных.**  
  Добавление `project_id` и новой таблицы `projects` проще всего сделать через пересоздание файла `database.db`.  
  Если вы не удалите старую базу, новая схема может не примениться, и SQL‑запросы начнут падать.

- **Временное упрощение прав доступа.**  
  Пока доступ к проекту и его заявкам есть только у **владельца**.  
  В Части H мы добавим участников и роли (`owner`, `maintainer`, `reporter`) и расширим логику допуска.

Эти изменения сознательные и не мешают дальнейшим частям — наоборот, они готовят почву для реализации ролей, комментариев и истории, описанных в `issue_parts_plan_v2.md` и `ISSUE_FINAL.md`.
{% endraw %}

