{% raw %}
# Часть I: Комментарии к заявкам

В Частях F–H вы уже сделали:

- базовый каркас приложения, пользователей, админку и общий макет (`GEN_PART_A`–`E`);
- первую версию заявок (`tickets`) и страницы:
  - список заявок проекта,
  - создание заявки,
  - детали заявки;
- проекты (`projects`) и привязку заявок к проектам (`project_id`) — заявки живут только внутри проекта;
- участников проектов (`project_members`) и роли:
  - `owner`, `maintainer`, `reporter`,
  - проверки доступа через `project_members.py`,
  - раздел «Участники проекта» на странице проекта,
  - разделение `/projects` на «Мои проекты» и «Проекты, где я участвую».

В этой части мы добавим **комментарии к заявкам**:

- таблицу `comments` в базе;
- загрузку комментариев в деталях заявки;
- форму добавления комментария;
- базовые правила, кто может комментировать (по участию в проекте).

> Важно: логика **архивации проекта** и запрета комментариев в архивных проектах
> появится только в Части J. Здесь мы заранее готовим место под такую проверку,
> но фактическая блокировка архива появится позже.

---

## I0. Актуальное состояние перед началом

Перед выполнением этой части у вас уже должны быть:

- таблицы `users`, `settings`, `projects`, `tickets`, `project_members` в `db.py`;
- модуль `project_members.py` с функциями:
  - `get_project_role(project_id)`,
  - `can_view_project(project_id)`,
  - `can_manage_members(project_id)`,
  - `can_create_ticket(project_id)`,
  - `can_edit_ticket_in_project(ticket_row)`;
- модуль `views_projects.py` с:
  - `projects_list_view`,
  - `project_new_view`, `project_create_view`,
  - `project_detail_view(project_id)`, который:
    - пускает только участников проекта,
    - загружает заявки проекта,
    - загружает участников проекта (`members`),
    - подготавливает `candidate_users` для выпадающего списка добавления участника;
- модуль `views_tickets.py` с:
  - `ticket_list_view(project_id)`,
  - `ticket_new_view(project_id)`,
  - `ticket_create_view(project_id)`,
  - `ticket_detail_view(project_id, ticket_id)`,
  - обновлёнными `can_view_ticket(ticket_id)` и `can_edit_ticket(ticket_id)`,
    которые используют проектные роли;
- шаблон `ticket_detail.html`, который пока показывает только данные заявки (без комментариев);
- маршруты в `app.py` вида:
  - `/projects/<int:project_id>/tickets`,
  - `/projects/<int:project_id>/tickets/new`,
  - `/projects/<int:project_id>/tickets/<int:ticket_id>`.

В Части I мы:

- **добавим** таблицу `comments`,
- **обновим** `views_tickets.py`, чтобы детали заявки подгружали комментарии и обрабатывали добавление,
- **расширим** `ticket_detail.html` секцией «Комментарии».

> Рекомендация: если вы меняете схему БД (добавляете таблицу `comments`) и недавно
> пересоздавали базу для Частей G и H, проще всего снова остановить сервер, удалить
> `database.db` и заново запустить приложение, чтобы `init_db()` создала все таблицы
> с учётом новых изменений.

---

## I1. Таблица `comments` в `db.py`

Начнём с схемы БД.

### I1.1. Добавляем таблицу `comments`

Откройте `db.py` и найдите функцию:

```python
def init_db():
    ...
```

Сразу **после** создания таблицы `tickets` (или рядом с ней) добавьте еще один `conn.execute`:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS comments (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            ticket_id INTEGER NOT NULL,
            user_id INTEGER,
            body TEXT NOT NULL,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            FOREIGN KEY (ticket_id) REFERENCES tickets(id),
            FOREIGN KEY (user_id) REFERENCES users(id)
        )
    """)
```

Пояснения:

- `ticket_id` — ссылка на заявку в таблице `tickets` (в плане это называлось `issue_id`,
  но в нашем проекте таблица называется `tickets`, поэтому мы используем название поля `ticket_id`);
- `user_id` — автор комментария (может быть `NULL`, если когда‑нибудь решите делать системные комментарии);
- `body` — текст комментария;
- `created_at` — время создания комментария, в том же формате UTC, что и в других таблицах.

После этого сохраните `db.py`. При следующем запуске приложения таблица будет создана.

---

## I2. Логика комментариев во `views_tickets.py`

Теперь научим страницу деталей заявки подгружать комментарии и добавлять новые.

Откройте `views_tickets.py`.

### I2.1. Импортируем `can_view_project`

В начале файла у вас уже есть импорты:

```python
from db import get_conn
from auth_utils import is_logged_in, current_user, is_admin
from project_members import can_view_project, can_edit_ticket_in_project, can_create_ticket
```

Если `can_view_project` ещё не импортирован — добавьте его в список импортов из `project_members`.

### I2.2. Загружаем комментарии в `ticket_detail_view`

Найдите функцию:

```python
def ticket_detail_view(project_id: int, ticket_id: int):
    ...
```

Сейчас внутри она:

- проверяет логин,
- вызывает `can_view_ticket(ticket_id)` и при отсутствии прав даёт `abort(404)`,
- загружает заявку `t` с полями из таблицы `tickets` и связанными пользователями,
- проверяет, что `t["project_id"]` совпадает с `project_id`,
- рендерит шаблон `ticket_detail.html`, передавая:
  - `ticket=t`,
  - `can_edit=can_edit_ticket(ticket_id)`,
  - `project_id=project_id`.

Обновите эту функцию, чтобы она также загружала комментарии и вычисляла флаг `can_comment`.

Пример (фрагмент внутри функции, после выборки `t` и проверки `project_id`):

```python
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

Пояснения:

- `comments` — список комментариев к этой заявке;
- `author_username` может быть `NULL`, если комментарий оставлен пользователем, которого потом удалили;
- `can_comment` сейчас означает:
  - пользователь залогинен,
  - пользователь является участником проекта (`can_view_project(project_id)`).

> В Части J к этому условию можно будет добавить проверку `is_archived` проекта,
> чтобы запретить добавление комментариев к архивным проектам.

### I2.3. Обработчик `comment_create_view`

Теперь добавим новый обработчик, который будет принимать POST с формой комментария.

В `views_tickets.py` ниже `ticket_detail_view` создайте функцию:

```python
def comment_create_view(project_id: int, ticket_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    # Проверяем, что пользователь вообще может видеть заявку и проект
    can_view, _ = can_view_ticket(ticket_id)
    if not can_view:
        abort(404)

    body = (request.form.get("body") or "").strip()
    if not body:
        # Пустые комментарии не сохраняем, просто возвращаемся на страницу заявки
        return redirect(url_for("ticket_detail", project_id=project_id, ticket_id=ticket_id))

    conn = get_conn()
    user = current_user()
    user_id = user["id"] if user is not None else None

    conn.execute(
        """
        INSERT INTO comments (ticket_id, user_id, body)
        VALUES (?, ?, ?)
        """,
        (ticket_id, user_id, body),
    )
    conn.commit()
    conn.close()

    return redirect(url_for("ticket_detail", project_id=project_id, ticket_id=ticket_id))
```

Это простая реализация:

- если пользователь не залогинен — отправляем на логин;
- если нет прав видеть заявку — 404;
- пустые комментарии игнорируем (редирект без сохранения);
- при валидном `body` создаём строку в `comments` и возвращаемся на страницу заявки.

> Если вы хотите показывать сообщение об ошибке при пустом комментарии,
> можно вместо прямого `redirect` сохранить текст ошибки во `flash` или передавать
> отдельный параметр в шаблон. Для учебной версии достаточно простого игнорирования.

---

## I3. Маршрут комментариев в `app.py`

Теперь нужно добавить маршрут, который будет вызывать `comment_create_view`.

Откройте `app.py`.

### I3.1. Импортируем обработчик из `views_tickets`

В блоке:

```python
from views_tickets import (
    ticket_list_view,
    ticket_new_view,
    ticket_create_view,
    ticket_detail_view,
)
```

добавьте `comment_create_view`:

```python
from views_tickets import (
    ticket_list_view,
    ticket_new_view,
    ticket_create_view,
    ticket_detail_view,
    comment_create_view,
)
```

### I3.2. Добавляем маршрут

Ниже уже существующего маршрута деталей заявки:

```python
@app.get("/projects/<int:project_id>/tickets/<int:ticket_id>")
def ticket_detail(project_id, ticket_id):
    return ticket_detail_view(project_id, ticket_id)
```

добавьте новый маршрут для POST:

```python
@app.post("/projects/<int:project_id>/tickets/<int:ticket_id>/comments")
def comment_create(project_id, ticket_id):
    return comment_create_view(project_id, ticket_id)
```

URL соответствует текущей схеме:

- всё живёт под `/projects/<project_id>/tickets/<ticket_id>...`.

---

## I4. Обновляем `ticket_detail.html`: секция комментариев

Осталось научить шаблон отображать список комментариев и форму добавления.

Откройте `templates/ticket_detail.html`.

Сейчас он выглядит (после Частей F–H) примерно так:

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

    <p>
      <strong>Статус:</strong> {{ ticket.status }}<br />
      <strong>Приоритет:</strong> {{ ticket.priority }}
    </p>

    <p>
      <strong>Автор:</strong> {{ ticket.reporter_username or '—' }}<br />
      <strong>Создана:</strong> {{ ticket.created_at }}
    </p>

    <h3>Описание</h3>
    <pre>{{ ticket.description }}</pre>
  </section>
{% endblock %}
```

### I4.1. Добавляем блок «Комментарии»

Под секцией с описанием (но внутри `{% block content %}`) добавьте новую секцию:

```html
  <section class="card" style="margin-top: 16px;">
    <h3>Комментарии</h3>

    {% if not comments %}
      <p class="muted">Пока нет ни одного комментария.</p>
    {% else %}
      <ul class="comments">
        {% for c in comments %}
          <li class="comment-item">
            <div class="comment-meta">
              <strong>{{ c.author_username or 'System' }}</strong>
              <span class="comment-date">
                {{ c.created_at[:19] if c.created_at else '' }}
              </span>
            </div>
            <pre class="comment-body">{{ c.body }}</pre>
          </li>
        {% endfor %}
      </ul>
    {% endif %}

    {% if can_comment %}
      <hr />
      <h4>Добавить комментарий</h4>

      <form
        method="post"
        action="{{ url_for('comment_create', project_id=project_id, ticket_id=ticket.id) }}"
        class="form"
      >
        <div class="form-group">
          <label for="body">Текст комментария</label>
          <textarea id="body" name="body" rows="3" required></textarea>
        </div>

        <button type="submit" class="btn btn-primary">Отправить</button>
      </form>
    {% else %}
      <p class="muted" style="margin-top: 8px;">
        Чтобы оставить комментарий, нужно быть участником проекта.
      </p>
    {% endif %}
  </section>
```

Пояснения:

- `comments` и `can_comment` приходят из `ticket_detail_view`;
- `author_username` может быть `NULL`, тогда показываем `'System'`;
- пока мы не проверяем архивность проекта, просто сообщаем, что комментировать могут только участники.

> В Части J вы сможете дополнить условие `can_comment` и шаблон так, чтобы
> при `project.is_archived` форма скрывалась, а вместо неё показывалась
> надпись вроде «Проект в архиве, комментарии только для чтения».

---

## I5. Видимая / проверяемая функциональность

После выполнения Части I проверьте поведение пошагово.

1. **Подготовка.**
   - При необходимости удалите `database.db`.
   - Запустите приложение: `python app.py`.
   - Убедитесь, что ошибок при создании таблиц, включая `comments`, нет.

2. **Создайте проект и заявку.**
   - Войдите под мастером или обычным пользователем.
   - Перейдите в «Проекты», создайте новый проект.
   - Откройте проект и создайте в нём заявку.
   - Откройте страницу детали заявки `/projects/<project_id>/tickets/<ticket_id>`.

3. **Проверьте, что блок «Комментарии» отображается.**
   - На странице заявки должен быть раздел «Комментарии».
   - При отсутствии комментариев виден текст «Пока нет ни одного комментария.»
     и форма добавления (если вы участник проекта).

4. **Добавление комментария.**
   - Введите текст комментария и отправьте форму.
   - После редиректа новый комментарий должен появиться в списке с вашим логином и временем.
   - Повторите с несколькими комментариями — порядок должен быть по времени (снизу или сверху согласно шаблону).

5. **Комментарии от другого участника.**
   - Добавьте другого пользователя в проект (как в Части H).
   - Войдите под этим пользователем, откройте ту же заявку.
   - Убедитесь, что:
     - пользователь видит все уже оставленные комментарии;
     - он может добавить свой комментарий, который также появляется в списке.

6. **Попытка комментировать без прав.**
   - Попробуйте открыть URL заявки для проекта, где вы **не участник**.
   - Ожидаемое поведение:
     - страница заявки должна быть недоступна (403/404 через уже существующую проверку `can_view_ticket` / `can_view_project`),
     - соответственно, форма добавления комментария недостижима.

---

## I6. Замечания и возможные конфликты

- **Имя внешнего ключа.**  
  В плане (`issue_parts_plan_v2.md`) поле связывалось как `issue_id`, но в вашей реализации
  основная таблица называется `tickets`. В Части I мы сознательно используем поле
  `ticket_id` с внешним ключом `FOREIGN KEY (ticket_id) REFERENCES tickets(id)`.

- **Архивирование проектов.**  
  Проверки `is_archived` для проектов на этом шаге ещё нет (она появится в Части J).
  В Части I мы ограничиваемся проверкой:
  - пользователь участник проекта,
  - пользователь залогинен.  
  Позже вы сможете дополнить логику `can_comment` и шаблон `ticket_detail.html`,
  чтобы в архивных проектах комментарии стали только для чтения.

- **Старые данные.**  
  Если вы не удалите старый `database.db`, таблица `comments` может не создаться
  автоматически, и попытка вставить комментарий приведёт к ошибке SQL.
  Поэтому для учебного потока удобнее явно пересоздавать базу при добавлении новых таблиц.

Эти изменения делают заявки «живыми» — у каждой появляется простая дискуссия.
В Части J вы сможете расширить это журналом истории и жизненным циклом проекта.

{% endraw %}

