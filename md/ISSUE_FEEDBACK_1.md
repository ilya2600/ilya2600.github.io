## Критические функциональные ошибки (которые приводят к падениям) и как исправить

Этот документ объясняет **только функциональные проблемы**, из‑за которых страницы проекта/заявок **падают с ошибками** (например, `TypeError`, `KeyError`, `Undefined` переменные, неверная генерация ссылок в шаблонах).

> Важно: вносите правки в файлы ровно в тех местах, которые указаны ниже. После каждого блока можно запускать приложение и проверять соответствующую страницу.

---

## 1) Страница детали проекта падает из-за `project`/`conn` (файл `views_projects.py`)

### Симптомы
На странице проекта (роут `/projects/<project_id>`) вы можете увидеть:
- `UnboundLocalError: local variable 'conn' referenced before assignment`
- `NameError: name 'project' is not defined`
- или другая ошибка, связанная с тем, что данные для отрисовки не подготовлены.

### Причина
В функции `project_detail_view(project_id)` в `views_projects.py`:
- строка `conn = get_conn()` находится **не на том уровне отступа**, из-за чего `conn` не создаётся в “нормальном” сценарии;
- в шаблон `project_detail.html` передаётся переменная `project=project`, но **переменная `project` не создаётся** (нет `project = get_project(project_id)`).

### Как исправить (копируйте и вставляйте)
Откройте файл `views_projects.py` и найдите функцию:
`def project_detail_view(project_id):`

Замените тело функции `project_detail_view` на следующую версию (сохраняйте `def ...` как есть, замените код внутри):

```python
def project_detail_view(project_id):
    """Страница одного проекта с его заявками."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    if user is None:
        return redirect(url_for("login_form"))

    # ВАЖНО: проект должен быть загружен, иначе ниже будет project=project (не определено)
    project = get_project(project_id)
    if project is None:
        abort(404)

    conn = get_conn()

    # Заявки проекта
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

    # Участники проекта
    members = conn.execute(
        """
        SELECT
            pm.user_id,
            pm.role,
            u.username,
            u.archived_at
        FROM project_members pm
        JOIN users u ON pm.user_id = u.id
        WHERE pm.project_id = ?
        ORDER BY
            CASE pm.role
                WHEN 'owner' THEN 0
                WHEN 'maintainer' THEN 1
                ELSE 2
            END,
            u.username
        """,
        (project_id,),
    ).fetchall()

    candidate_users = conn.execute(
        """
        SELECT id, username
        FROM users
        WHERE archived_at IS NULL
          AND id NOT IN (
            SELECT user_id FROM project_members WHERE project_id = ?
          )
        ORDER BY username
        """,
        (project_id,),
    ).fetchall()

    conn.close()

    return render_template(
        "project_detail.html",
        project=project,
        tickets=tickets,
        members=members,
        project_role=get_project_role(project_id),
        candidate_users=candidate_users,
    )
```

### Что именно важно запомнить
- `project = get_project(project_id)` должен быть **до** `render_template(... project=project ...)`.
- `conn = get_conn()` должен быть **с правильными отступами**, чтобы выполнялся, когда `user is not None`.

---

## 2) Страница списка заявок падает из‑за несовпадения аргументов (файл `views_tickets.py`)

### Симптом
- `TypeError: tickets_for_project() missing 1 required positional argument: 'user'`
- или похожая ошибка.

### Причина
В `views_tickets.py` функция `tickets_for_project` объявлена как:
`def tickets_for_project(project_id, user):`
а в `ticket_list_view` вызывается как:
`tickets_for_project(project_id)`

Плюс внутри используется `user["is_admin"]`, но в таблице `users` роль хранится в поле `role` (то есть ключа `is_admin` там нет).

### Как исправить (копируйте и вставляйте)
Откройте `views_tickets.py`.

#### 2.1. Исправьте функцию `tickets_for_project`
Замените всю функцию `tickets_for_project(...)` на следующую версию:

```python
def tickets_for_project(project_id):
    """Вернуть все заявки этого проекта с учетом роли текущего пользователя (admin/не admin)."""
    user = current_user()
    if user is None:
        return []

    conn = get_conn()
    try:
        if is_admin():
            # Админ/менеджер видит все заявки проекта
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
        else:
            # Остальные видят только заявки, где они автор или исполнитель
            # (тут логика может отличаться от требований, но это НЕ про падение — это про “какие заявки покажут”)
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
                    ru.archived_at AS reporter_archived,
                    au.username AS assignee_username,
                    au.archived_at AS assignee_archived
                FROM tickets t
                LEFT JOIN users ru ON t.reporter_id = ru.id
                LEFT JOIN users au ON t.assignee_id = au.id
                WHERE (t.reporter_id = ? OR t.assignee_id = ?)
                  AND t.project_id = ?
                ORDER BY t.created_at DESC
                """,
                (user["id"], user["id"], project_id),
            ).fetchall()

        return rows
    finally:
        conn.close()
```

#### 2.2. Проверьте `ticket_list_view`
После изменения сигнатуры `tickets_for_project(project_id)` ваш текущий код:
`tickets = tickets_for_project(project_id)`
должен работать.

---

## 3) Страница новой заявки падает из-за `get_project` (файл `views_tickets.py`)

### Симптом
- `NameError: name 'get_project' is not defined`

### Причина
В `ticket_new_view(project_id)` есть строка:
`project = get_project(project_id)`
но в `views_tickets.py` функция `get_project` не импортирована/не определена.

### Как исправить
В верхней части `views_tickets.py` добавьте импорт:

```python
from views_projects import get_project
```

Обычно он ставится рядом с остальными `from ... import ...`.

После этого `ticket_new_view` сможет вызвать `get_project(project_id)`.

---

## 4) Детальная страница заявки падает из‑за сигнатуры + отсутствия `project_id` в запросе (файл `views_tickets.py`)

### Симптомы
Вы можете увидеть одно или несколько:
- `TypeError: ticket_detail_view() takes 1 positional argument but 2 were given`
- `NameError: name 'project_id' is not defined`
- `KeyError: 'project_id'`

### Причины
1) В `app.py` роут вызывает:
`return ticket_detail_view(project_id, ticket_id)`
но функция объявлена как:
`def ticket_detail_view(ticket_id: int):`

2) Внутри `ticket_detail_view` используется переменная `project_id`, но её нет в параметрах функции (после вызова падает как минимум по несоответствию аргументов/переменной).

3) SQL в `ticket_detail_view` не выбирает `t.project_id`, но дальше есть проверка:
`if t is None or t["project_id"] != project_id: ...`
Это приводит к `KeyError: 'project_id'`.

### Как исправить (копируйте и вставляйте)
В `views_tickets.py` замените функцию `ticket_detail_view` на следующую версию:

```python
def ticket_detail_view(project_id: int, ticket_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    can_view, _ = can_view_ticket(ticket_id)
    if not can_view:
        abort(404)

    conn = get_conn()
    t = conn.execute("""
        SELECT
            t.id,
            t.project_id,           -- ВАЖНО: нужен для проверки и отладки
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

    if t is None or t["project_id"] != project_id:
        abort(404)

    conn = get_conn()
    comments = conn.execute(
        """
        SELECT
            c.id,
            c.body,
            c.created_at,
            c.user_id,
            u.username AS author_username
        FROM comments c
        LEFT JOIN users u ON c.user_id = u.id
        WHERE c.ticket_id = ?
        ORDER BY c.created_at ASC
        """,
        (ticket_id,),
    ).fetchall()
    conn.close()

    user = current_user()
    can_comment = (
        user is not None
        and can_view_project(project_id)
    )

    return render_template(
        "ticket_detail.html",
        ticket=t,
        can_edit=can_edit_ticket(ticket_id),
        project_id=project_id,
        comments=comments,
        can_comment=can_comment,
    )
```

### Что важно проверить после вставки
- Сигнатура теперь: `ticket_detail_view(project_id, ticket_id)` (строго в таком порядке, как вызывает `app.py`).
- В SQL добавлено: `t.project_id`.
- В `render_template` передаётся `project_id=project_id` (нужно для шаблона `ticket_detail.html`).

---

## 5) Шаблон `ticket_detail.html` содержит битую ссылку на список заявок (файл `templates/ticket_detail.html`)

### Симптом
На странице детали заявки при рендеринге может возникать ошибка уровня шаблона вида:
- `BuildError: Could not build url for endpoint 'ticket_list' with values ...`

### Причина
В `templates/ticket_detail.html` ссылка выглядит так:

```jinja2
<a href="{{ url_for('ticket_list') }}">К списку заявок</a>
```

Но роут `ticket_list` требует параметр `project_id`:
`@app.get("/projects/<int:project_id>/tickets")`

Значит, нужно передать `project_id`.

### Как исправить (одна строка)
В `templates/ticket_detail.html` найдите строку с:
`url_for('ticket_list')`

Замените на:

```jinja2
<a href="{{ url_for('ticket_list', project_id=project_id) }}">К списку заявок</a>
```

---

## Как быстро проверить, что всё починили
1. Запустите приложение и откройте страницу:
   - `/projects` (выберите проект)
2. Откройте:
   - страницу конкретного проекта `/projects/<id>`
3. Внутри проекта откройте:
   - список заявок `/projects/<project_id>/tickets`
   - страницу новой заявки `/projects/<project_id>/tickets/new`
   - детальную страницу заявки `/projects/<project_id>/tickets/<ticket_id>`
4. На странице заявки убедитесь, что:
   - комментарии отображаются/страница не падает
   - ссылка “К списку заявок” работает и возвращает туда же (с `project_id`).

