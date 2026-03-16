{% raw %}
# Часть H: Участники проектов и роли

В Части G вы:

- добавили таблицу `projects` в `db.py`,
- расширили таблицу `tickets`, добавив `project_id`,
- создали модуль `views_projects.py` и страницы:
  - список проектов `/projects`,
  - создание `/projects/new`,
  - просмотр `/projects/<id>` с блоком «Заявки этого проекта»,
- перенесли заявки под URL вида:
  - `/projects/<project_id>/tickets`,
  - `/projects/<project_id>/tickets/new`,
  - `/projects/<project_id>/tickets/<ticket_id>`,
- обновили шаблоны заявок так, чтобы они работали **внутри проекта**, а не глобально.

Пока что:

- доступ к проектам и заявкам есть только у **владельца проекта** (плюс глобального администратора через общие права),
- нет таблицы участников проекта,
- нет **проектных ролей** (`owner`, `maintainer`, `reporter`),
- нет разделения на «Мои проекты» и «Проекты, где я участвую».

В этой части мы:

- добавим таблицу `project_members`,
- введём проектные роли и вспомогательные функции,
- расширим страницу проекта разделом «Участники»,
- научим владельца приглашать других пользователей и задавать им роли,
- разделим список проектов на «Мои» и «Где я участвую»,
- обновим права доступа к проектам и заявкам так, чтобы они опирались на membership.

> Важно: мы **не вводим ещё комментарии, историю и архивирование проектов** — это задачи частей I и J.  
> Здесь мы сосредоточимся исключительно на **membership и ролях**.

---

## H0. Актуальное состояние перед началом

Перед выполнением этой части у вас уже должны быть:

- базовая система пользователей и ролей (`user`, `admin`) из частей B–E (`auth_utils.py`, `views_auth.py`, `views_admin.py`);
- таблицы `settings`, `users`, `tickets`, `projects` в `db.py`;
- модуль `views_projects.py` со страницами:
  - `projects_list_view` (список проектов, где текущий пользователь владелец),
  - `project_new_view`, `project_create_view`,
  - `project_detail_view(project_id)` — пока пускает только владельца;
- модуль `views_tickets.py`, в котором:
  - заявки уже привязаны к проекту через `project_id`,
  - маршруты в `app.py` имеют вид `/projects/<project_id>/tickets/...`,
  - функции `ticket_list_view(project_id)`, `ticket_new_view(project_id)`,  
    `ticket_create_view(project_id)`, `ticket_detail_view(project_id, ticket_id)`
    принимают `project_id` и работают только внутри одного проекта;
  - хелперы `can_view_ticket(ticket_id)` и `can_edit_ticket(ticket_id)`  
    пока ещё используют базовую логику из Части F (админ видит всё, обычный пользователь — свои заявки).

В Части H мы:

- **не меняем** схему `tickets` и `projects`,
- **добавляем** новую таблицу `project_members`,
- **перенастраиваем** доступ к проектам и заявкам, чтобы он опирался на membership.

> Рекомендация: после добавления таблицы `project_members` и изменения логики доступа
> лучше остановить сервер, удалить `database.db` и заново запустить приложение, чтобы
> база создалась с новой схемой, а проекты и участники были созданы уже по новым правилам.

---

## H1. Таблица `project_members` в `db.py`

Начнём с таблицы, которая будет хранить участников и их роли в проектах.

Откройте файл `db.py` и найдите функцию:

```python
def init_db():
    ...
```

### H1.1. Создаём таблицу `project_members`

Сразу **после** создания таблицы `projects` (и до `tickets`) добавьте новый `conn.execute`:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS project_members (
            project_id INTEGER NOT NULL,
            user_id INTEGER NOT NULL,
            role TEXT NOT NULL CHECK (role IN ('owner', 'maintainer', 'reporter')),
            PRIMARY KEY (project_id, user_id),
            FOREIGN KEY (project_id) REFERENCES projects(id),
            FOREIGN KEY (user_id) REFERENCES users(id)
        )
    """)
```

Пояснения:

- `project_id`, `user_id` вместе образуют **составной первичный ключ** — один пользователь может иметь только одну роль в проекте,
- `role` ограничена набором `'owner'`, `'maintainer'`, `'reporter'`,
- внешние ключи связывают записи с реальными проектами и пользователями.

### H1.2. Автоматическое добавление владельца в `project_members`

Теперь нужно, чтобы при создании проекта его владелец **автоматически** появлялся в `project_members` с ролью `owner`.

Откройте `views_projects.py` и найдите функцию:

```python
def project_create_view():
    ...
```

Внутри у вас уже есть код, который:

- берёт текущего пользователя `user = current_user()`,
- валидирует заголовок,
- вставляет строку в таблицу `projects`:

```python
    conn = get_conn()
    conn.execute(
        "INSERT INTO projects (owner_id, title, description) VALUES (?, ?, ?)",
        (user["id"], title, description),
    )
    conn.commit()
    conn.close()
```

Обновите эту часть так, чтобы:

1. сначала вставить проект,
2. получить его `id` через `cursor.lastrowid`,
3. записать в `project_members` владельца с ролью `'owner'`.

Пример (фрагмент внутри `project_create_view`):

```python
    conn = get_conn()
    cursor = conn.execute(
        "INSERT INTO projects (owner_id, title, description) VALUES (?, ?, ?)",
        (user["id"], title, description),
    )
    project_id = cursor.lastrowid

    # Владелец проекта автоматически становится участником с ролью owner
    conn.execute(
        "INSERT INTO project_members (project_id, user_id, role) VALUES (?, ?, 'owner')",
        (project_id, user["id"]),
    )

    conn.commit()
    conn.close()
```

> На этом этапе в каждом проекте гарантированно есть хотя бы один участник — владелец (`owner`).

---

## H2. Новый модуль `project_members.py` с хелперами ролей

Чтобы не засорять `views_projects.py` и `views_tickets.py`, вынесем проверки ролей в отдельный модуль.

### H2.1. Создаём файл

В корне проекта (рядом с `app.py`, `views_projects.py`, `views_tickets.py`) создайте файл:

- `project_members.py`

### H2.2. Базовые импорты и `get_project_role`

Откройте `project_members.py` и вставьте:

```python
from db import get_conn
from auth_utils import current_user, is_admin


def get_project_role(project_id):
    """Вернуть роль текущего пользователя в проекте или None, если он не участник.

    Администратор (master) считается владельцем всех проектов.
    """
    user = current_user()
    if user is None:
        return None

    if is_admin():
        # Глобальный администратор обладает максимальными правами
        return "owner"

    conn = get_conn()
    row = conn.execute(
        """
        SELECT role
        FROM project_members
        WHERE project_id = ? AND user_id = ?
        """,
        (project_id, user["id"]),
    ).fetchone()
    conn.close()

    if row is None:
        return None

    return row["role"]
```

### H2.3. Хелперы `can_view_project` и `can_manage_members`

Добавьте в этот же файл:

```python
def can_view_project(project_id):
    """Можно ли текущему пользователю видеть этот проект."""
    role = get_project_role(project_id)
    return role is not None


def can_manage_members(project_id):
    """Можно ли управлять участниками проекта (добавлять/удалять, менять роли)."""
    role = get_project_role(project_id)
    return role == "owner"
```

### H2.4. Хелперы прав для заявок

Части I и J будут расширять поведение заявок, но уже сейчас нам нужно опереться на роли:

- **создавать** заявки могут все участники (`owner`, `maintainer`, `reporter`),
- **редактировать** любые заявки могут `owner` и `maintainer`,
- **редактировать свои** заявки может `reporter`, если он автор заявки.

Добавьте в `project_members.py`:

```python
def can_create_ticket(project_id):
    """Можно ли создавать заявки в этом проекте."""
    role = get_project_role(project_id)
    return role in ("owner", "maintainer", "reporter")


def can_edit_ticket_in_project(ticket_row):
    """Можно ли редактировать конкретную заявку."""
    # ticket_row должен содержать как минимум поля:
    # project_id, reporter_id
    user = current_user()
    if user is None:
        return False

    if is_admin():
        return True

    role = get_project_role(ticket_row["project_id"])

    if role in ("owner", "maintainer"):
        return True

    if role == "reporter" and ticket_row["reporter_id"] == user["id"]:
        return True

    return False
```

> Эти функции мы будем использовать в `views_projects.py` и `views_tickets.py`, чтобы не размазывать логику по нескольким файлам.

---

## H3. Расширяем `project_detail_view`: раздел «Участники»

Теперь добавим на страницу проекта:

- список участников,
- форму добавления участника (только для владельца),
- возможность удалить участника (кроме владельца).

### H3.1. Получение участников проекта в `views_projects.py`

Откройте `views_projects.py` и в начало файла добавьте импорт:

```python
from project_members import (
    can_view_project,
    can_manage_members,
    get_project_role,
)
```

Далее найдите функцию:

```python
def project_detail_view(project_id):
    ...
```

Сейчас она:

- проверяет логин,
- загружает проект,
- пускает только владельца (`project["owner_id"] == user["id"]`),
- загружает список заявок `tickets` и рендерит `project_detail.html`.

Обновите проверки доступа так, чтобы они опирались на `can_view_project`:

```python
    project = get_project(project_id)
    if project is None:
        abort(404)

    # Доступ теперь по membership (владелец или участник проекта)
    if not can_view_project(project_id):
        abort(403)
```

Теперь добавим загрузку участников проекта:

```python
    conn = get_conn()

    # Заявки проекта (как в Части G)
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
```

И добавьте в `render_template`:

```python
    return render_template(
        "project_detail.html",
        project=project,
        tickets=tickets,
        members=members,
        project_role=get_project_role(project_id),
        candidate_users=candidate_users,
    )
```

Здесь:

- `members` — список всех участников проекта,
- `candidate_users` — активные пользователи, которых ещё нет в `project_members` для этого проекта,
- `project_role` — роль **текущего пользователя** в этом проекте (`owner` / `maintainer` / `reporter` / `None`).

### H3.2. Обработчики добавления и удаления участников

В `views_projects.py` ниже `project_detail_view` добавим две новые функции.

Импорты в начале файла уже должны содержать:

```python
from flask import render_template, redirect, url_for, request, abort, flash
from auth_utils import is_logged_in, current_user, is_admin
from project_members import (
    can_view_project,
    can_manage_members,
    get_project_role,
)
```

#### Добавление участника

```python
def project_member_add_view(project_id):
    """Добавить участника в проект (только владелец)."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    project = get_project(project_id)
    if project is None:
        abort(404)

    if not can_manage_members(project_id):
        abort(403)

    user_id = request.form.get("user_id")
    role = request.form.get("role") or ""

    if not user_id or role not in ("maintainer", "reporter"):
        flash("Некорректные данные участника.")
        return redirect(url_for("project_detail", project_id=project_id))

    try:
        user_id_int = int(user_id)
    except ValueError:
        flash("Некорректный пользователь.")
        return redirect(url_for("project_detail", project_id=project_id))

    conn = get_conn()

    # Проверим, что такой пользователь существует и не является архивированным
    u = conn.execute(
        "SELECT id, username, archived_at FROM users WHERE id = ?",
        (user_id_int,),
    ).fetchone()
    if u is None or u["archived_at"] is not None:
        conn.close()
        flash("Нельзя добавить несуществующего или архивированного пользователя.")
        return redirect(url_for("project_detail", project_id=project_id))

    # Не дублируем участников
    existing = conn.execute(
        """
        SELECT 1
        FROM project_members
        WHERE project_id = ? AND user_id = ?
        """,
        (project_id, user_id_int),
    ).fetchone()

    if existing is None:
        conn.execute(
            """
            INSERT INTO project_members (project_id, user_id, role)
            VALUES (?, ?, ?)
            """,
            (project_id, user_id_int, role),
        )
        conn.commit()
        flash(f"Пользователь {u['username']} добавлен в проект как {role}.")
    else:
        flash("Этот пользователь уже участвует в проекте.")

    conn.close()
    return redirect(url_for("project_detail", project_id=project_id))
```

#### Удаление участника

```python
def project_member_remove_view(project_id, user_id):
    """Удалить участника из проекта (кроме владельца)."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    project = get_project(project_id)
    if project is None:
        abort(404)

    if not can_manage_members(project_id):
        abort(403)

    try:
        user_id_int = int(user_id)
    except ValueError:
        abort(400)

    conn = get_conn()

    # Нельзя удалить владельца проекта
    member = conn.execute(
        """
        SELECT pm.role, u.username
        FROM project_members pm
        JOIN users u ON pm.user_id = u.id
        WHERE pm.project_id = ? AND pm.user_id = ?
        """,
        (project_id, user_id_int),
    ).fetchone()

    if member is None:
        conn.close()
        flash("Участник не найден.")
        return redirect(url_for("project_detail", project_id=project_id))

    if member["role"] == "owner":
        conn.close()
        flash("Нельзя удалить владельца проекта.")
        return redirect(url_for("project_detail", project_id=project_id))

    conn.execute(
        "DELETE FROM project_members WHERE project_id = ? AND user_id = ?",
        (project_id, user_id_int),
    )
    conn.commit()
    conn.close()

    flash(f"Пользователь {member['username']} удалён из проекта.")
    return redirect(url_for("project_detail", project_id=project_id))
```

### H3.3. Маршруты в `app.py` для управления участниками

Откройте `app.py` и рядом с импортами `views_projects` добавьте новые функции:

```python
from views_projects import (
    projects_list_view,
    project_new_view,
    project_create_view,
    project_detail_view,
    project_member_add_view,
    project_member_remove_view,
)
```

Ниже существующих маршрутов проектов добавьте:

```python
@app.post("/projects/<int:project_id>/members/add")
def project_member_add(project_id):
    return project_member_add_view(project_id)


@app.post("/projects/<int:project_id>/members/<int:user_id>/remove")
def project_member_remove(project_id, user_id):
    return project_member_remove_view(project_id, user_id)
```

---

## H4. Обновляем `project_detail.html`: блок «Участники проекта»

Теперь расширим шаблон проекта, чтобы:

- показывать участников,
- давать владельцу возможность добавлять/удалять их.

Откройте `templates/project_detail.html`.

В Части G в нём уже есть:

- заголовок с названием проекта,
- описание и дата создания,
- блок «Заявки этого проекта» с таблицей.

### H4.1. Добавляем секцию «Участники проекта»

Под существующим блоком «Заявки этого проекта» добавьте новый `<section>`:

```html
  <section class="card" style="margin-top: 16px;">
    <h3>Участники проекта</h3>

    {% if not members %}
      <p class="muted">В этом проекте пока нет участников, кроме владельца.</p>
    {% else %}
      <table class="table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Логин</th>
            <th>Роль</th>
            <th>Статус</th>
            <th>Действия</th>
          </tr>
        </thead>
        <tbody>
          {% for m in members %}
            <tr>
              <td>{{ m.user_id }}</td>
              <td>{{ m.username }}</td>
              <td>{{ m.role }}</td>
              <td>
                {% if m.archived_at %}
                  <span class="badge badge-deleted">В архиве</span>
                {% else %}
                  Активен
                {% endif %}
              </td>
              <td>
                {% if project_role == 'owner' and m.role != 'owner' %}
                  <form
                    method="post"
                    action="{{ url_for('project_member_remove', project_id=project.id, user_id=m.user_id) }}"
                    class="inline-form"
                  >
                    <button
                      type="submit"
                      class="btn btn-sm btn-danger"
                      onclick="return confirm('Удалить участника из проекта?')"
                    >
                      Удалить
                    </button>
                  </form>
                {% endif %}
              </td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    {% endif %}

    {% if project_role == 'owner' %}
      <hr />
      <h4>Добавить участника</h4>

      <form
        method="post"
        action="{{ url_for('project_member_add', project_id=project.id) }}"
        class="form form-row"
      >
        <div class="form-group">
          <label for="user_id">Пользователь</label>
          <select id="user_id" name="user_id" required>
            {% for u in candidate_users %}
              <option value="{{ u.id }}">{{ u.username }}</option>
            {% endfor %}
          </select>
        </div>

        <div class="form-group">
          <label for="role">Роль</label>
          <select id="role" name="role">
            <option value="maintainer">maintainer</option>
            <option value="reporter">reporter</option>
          </select>
        </div>

        <div class="form-group" style="align-self: flex-end;">
          <button type="submit" class="btn btn-primary">Добавить участника</button>
        </div>
      </form>

      <p class="muted" style="margin-top: 8px;">
        Владелец (owner) задаётся автоматически при создании проекта и не может быть удалён через этот интерфейс.
        Список в выпадающем меню показывает только активных пользователей, которые ещё не добавлены в проект.
      </p>
    {% endif %}
  </section>
```

Обратите внимание:

- переменная `members` приходит из `project_detail_view`,
- `project_role` определяет, видит ли текущий пользователь форму добавления и кнопки удаления:
  - только `owner` может управлять участниками;
- выпадающий список пользователей строится на основе `candidate_users`, которые передаёт `project_detail_view`.

---

## H5. Обновляем список проектов: «Мои» и «Где я участвую»

Сейчас `/projects` показывает только проекты, где текущий пользователь является `owner` (через поле `owner_id` в таблице `projects`).

После появления `project_members` логичнее:

- «Мои проекты» — проекты, где текущий пользователь имеет роль `owner`,
- «Проекты, где я участвую» — проекты, где его роль `maintainer` или `reporter`.

### H5.1. Обновляем `projects_list_view` в `views_projects.py`

Откройте `views_projects.py` и найдите:

```python
def projects_list_view():
    """Список проектов текущего пользователя (как владелец)."""
    ...
```

Замените SQL-запросы на два отдельных:

```python
    conn = get_conn()

    # Проекты, где я владелец
    my_projects = conn.execute(
        """
        SELECT p.id, p.title, p.description, p.is_archived, p.created_at
        FROM projects p
        JOIN project_members pm
          ON pm.project_id = p.id
        WHERE pm.user_id = ? AND pm.role = 'owner' AND p.is_archived = 0
        ORDER BY p.created_at DESC
        """,
        (user["id"],),
    ).fetchall()

    # Проекты, где я участник (но не владелец)
    collaborating_projects = conn.execute(
        """
        SELECT p.id, p.title, p.description, p.is_archived, p.created_at, pm.role
        FROM projects p
        JOIN project_members pm
          ON pm.project_id = p.id
        WHERE pm.user_id = ?
          AND pm.role IN ('maintainer', 'reporter')
          AND p.is_archived = 0
        ORDER BY p.created_at DESC
        """,
        (user["id"],),
    ).fetchall()

    conn.close()

    return render_template(
        "project_list.html",
        my_projects=my_projects,
        collaborating_projects=collaborating_projects,
    )
```

### H5.2. Обновляем `project_list.html` под два списка

Откройте `templates/project_list.html`.

Замените содержимое блока `{% block content %}` на вариант с двумя секциями:

```html
{% block content %}
  <section class="card">
    <h2>Мои проекты</h2>

    <p>
      <a href="{{ url_for('project_new') }}" class="btn btn-primary">Новый проект</a>
    </p>

    {% if not my_projects %}
      <p class="muted">У вас пока нет собственных проектов.</p>
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
          {% for p in my_projects %}
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

  <section class="card" style="margin-top: 16px;">
    <h2>Проекты, где я участвую</h2>

    {% if not collaborating_projects %}
      <p class="muted">Вы пока не добавлены ни в один чужой проект.</p>
    {% else %}
      <table class="table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Название</th>
            <th>Моя роль</th>
            <th>Создан</th>
          </tr>
        </thead>
        <tbody>
          {% for p in collaborating_projects %}
            <tr>
              <td>
                <a href="{{ url_for('project_detail', project_id=p.id) }}">{{ p.id }}</a>
              </td>
              <td>
                <a href="{{ url_for('project_detail', project_id=p.id) }}">{{ p.title }}</a>
              </td>
              <td>{{ p.role }}</td>
              <td>{{ p.created_at[:10] if p.created_at else '' }}</td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    {% endif %}
  </section>
{% endblock %}
```

Теперь:

- владелец видит свои проекты в первой таблице,
- участник (maintainer/reporter) — во второй,
- администратор (`is_admin() == True`) через `get_project_role` автоматически считается `owner` для всех проектов.

---

## H6. Права на заявки: опираемся на project_members

Части F и G уже добавили в `views_tickets.py` хелперы:

- `can_view_ticket(ticket_id)`,
- `can_edit_ticket(ticket_id)`,

которые пока:

- дают админу полный доступ,
- обычному пользователю — доступ только к своим заявкам (как репортёр или исполнитель).

После Части G все заявки живут **внутри проектов**, поэтому логично:

- видеть и создавать заявки могут только участники соответствующего проекта,
- редактировать — согласно проектным ролям.

### H6.1. Обновляем `can_view_ticket` в `views_tickets.py`

Откройте `views_tickets.py` и добавьте в начало файла импорт:

```python
from project_members import can_view_project, can_edit_ticket_in_project
```

Найдите функцию `can_view_ticket(ticket_id)` и замените её тело на логику, которая:

- проверяет, существует ли заявка,
- берёт `project_id` из заявки,
- использует `can_view_project(project_id)` для проверки доступа.

Пример:

```python
def can_view_ticket(ticket_id):
    """Можно ли просматривать эту заявку текущему пользователю."""
    user = current_user()
    if user is None:
        return False, None

    conn = get_conn()
    t = conn.execute(
        """
        SELECT
            id,
            reporter_id,
            assignee_id,
            project_id
        FROM tickets
        WHERE id = ?
        """,
        (ticket_id,),
    ).fetchone()
    conn.close()

    if t is None:
        return False, None

    # Админ по‑прежнему видит все заявки
    if is_admin():
        return True, t

    # Остальные — только если они участники проекта
    if not can_view_project(t["project_id"]):
        return False, None

    return True, t
```

### H6.2. Обновляем `can_edit_ticket`

Найдите функцию `can_edit_ticket(ticket_id)` и обновите её так, чтобы она:

- загружала заявку вместе с `project_id` и `reporter_id`,
- вызывала `can_edit_ticket_in_project(ticket_row)`.

Пример:

```python
def can_edit_ticket(ticket_id):
    """Можно ли редактировать эту заявку (менять статус, исполнителя и т.п.)."""
    user = current_user()
    if user is None:
        return False

    if is_admin():
        return True

    conn = get_conn()
    t = conn.execute(
        """
        SELECT
            id,
            reporter_id,
            assignee_id,
            project_id
        FROM tickets
        WHERE id = ?
        """,
        (ticket_id,),
    ).fetchone()
    conn.close()

    if t is None:
        return False

    return can_edit_ticket_in_project(t)
```

### H6.3. Проверка `ticket_detail_view` и создания заявок

В Части G вы обновили:

- `ticket_detail_view(project_id, ticket_id)`,
- `ticket_create_view(project_id)`,

чтобы они принимали `project_id` и работали внутри проекта.

Убедитесь, что:

- в начале `ticket_detail_view` остаётся проверка:

  ```python
  can_view, _ = can_view_ticket(ticket_id)
  if not can_view:
      abort(404)
  ```

  — теперь она учитывает membership;

- `ticket_create_view(project_id)` допускает создание заявок только от участников:

  ```python
  from project_members import can_create_ticket

  ...

  def ticket_create_view(project_id):
      if not is_logged_in():
          return redirect(url_for("login_form", next=request.url))

      if not can_create_ticket(project_id):
          abort(403)

      ...
  ```

  (остальная логика валидации и вставки в БД остаётся как в Части G).

---

## H7. Видимая / проверяемая функциональность

После выполнения Части H вы сможете проверить следующую последовательность.

1. **Подготовка.**
   - Удалите `database.db` (при необходимости).
   - Запустите приложение (`python app.py`), чтобы создать новую схему с `project_members`.

2. **Создание проекта и участников.**
   - Залогиньтесь под мастером (`master` / `master`) или другим пользователем.
   - Перейдите в «Проекты» (ссылка в шапке).
   - Создайте новый проект.
   - Перейдите на страницу проекта `/projects/<id>`.
   - Внизу появится раздел «Участники проекта», в котором уже есть владелец.
   - Через форму «Добавить участника» добавьте другого пользователя как:
     - `maintainer`,
     - или `reporter`.

3. **Проверка списка проектов.**
   - Выйдите из аккаунта владельца.
   - Залогиньтесь под пользователем, которого вы добавили в проект.
   - Откройте `/projects`:
     - во второй секции «Проекты, где я участвую» вы увидите проект и свою роль;
     - клик по проекту ведёт на ту же страницу `/projects/<id>`.

4. **Проверка прав на заявки.**
   - На странице проекта зайдите в раздел «Заявки этого проекта».
   - Как участник (`maintainer` или `reporter`) создайте новую заявку:
     - кнопка «Новая заявка» ведёт на `/projects/<id>/tickets/new`,
     - после сохранения заявка появляется в списке.
   - Как владелец (`owner`):
     - открывайте и редактируйте любые заявки.
   - Как участник с ролью `reporter`:
     - можете создавать заявки и (позже, в следующих частях) менять статус **только своих** заявок;
     - редактирование чужих заявок будет запрещено хелперами прав.

5. **Удаление участника.**
   - Вернитесь под владельцем проекта.
   - На странице проекта в таблице участников нажмите «Удалить» для участника с ролью `maintainer` или `reporter`.
   - Убедитесь, что:
     - участник исчез из списка,
     - при входе под этим пользователем проект больше не отображается в секции «Проекты, где я участвую»,
     - попытка открыть `/projects/<id>` или `/projects/<id>/tickets/...` приводит к 403 или 404.

---

## H8. Замечания и возможные конфликты

- **Совместимость с Частью F.**  
  В Части F вы реализовали глобальную функцию `tickets_for_user()` и глобальный список `/tickets`.  
  После Части G эти маршруты были перенесены в пространство `/projects/<project_id>/tickets/...`, а логика отображения заявок стала **проектной**. В Части H мы полностью переориентируем права на **проектные роли**; глобальный список «моих заявок» может быть реализован отдельно позже (как дополнительное упражнение).

- **Роль администратора (`is_admin`).**  
  В базовых частях (B–E, F, G) администратор имел расширенные права в админ‑панели и при работе с заявками.  
  В Части H мы сохраняем это поведение: глобальный админ **всегда** может видеть и редактировать все проекты и заявки, независимо от membership, через хелперы `is_admin()` и `get_project_role`.

- **Статус архивирования проектов.**  
  Поле `is_archived` уже есть в таблице `projects` (добавлено в Части G), но логика архивирования/восстановления появится только в Части J.  
  В этой части мы фильтруем списки проектов по `is_archived = 0`, но не даём пользователю управления этим статусом.

- **Простота UI для выбора участника.**  
  В этой части мы используем **выпадающий список пользователей** для добавления участника в проект.
  Список формируется на стороне сервера на основе таблицы `users` и `project_members`:
  показываются только активные пользователи, которые ещё не участвуют в текущем проекте.  
  Это делает UX заметно лучше, но при этом сохраняет учебный акцент на таблице `project_members` и ролях.

Эти изменения подготавливают основу для Части I (комментарии к заявкам) и Части J (история изменений и жизненный цикл проекта), где права будут опираться на те же проектные роли и membership.

{% endraw %}

