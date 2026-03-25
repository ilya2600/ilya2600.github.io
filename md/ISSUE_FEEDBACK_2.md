# ISSUE_FEEDBACK_2

Ниже пошаговая инструкция по исправлению проблем из `PROJECT_ISSUES_AUDIT.md`.
Формат: **что сломано -> что вставить -> как проверить**.

---

## 0) Быстрый план выполнения

Делай в таком порядке:

1. Исправить `ticket_detail.html` (комментарии/история должны отображаться).
2. Исправить `comment_create_view` (лог истории `comment_added` + убрать мертвый код).
3. Добавить проверки доступа в `views_projects.py` и `views_tickets.py`.
4. Починить ссылки регистрации (`register_form` вместо `register` для GET-ссылок).
5. Доделать read-only для архивного проекта.
6. Починить кнопку восстановления проекта (route `project_restore`).
7. Убрать старые дубли в `static/app/`.
8. Перенести admin bootstrap-учетку в `.env`.

---

## 1) Комментарии и история не видны в карточке заявки

### Что сломано

В `templates/ticket_detail.html` блоки комментариев/истории находятся после `{% endblock %}`, поэтому не рендерятся.

### Что сделать

Весь контент страницы заявки (основной блок, комментарии, история) должен быть **внутри одного**:

```html
{% block content %}
...
{% endblock %}
```

### Готовый шаблон (можно вставить целиком в `templates/ticket_detail.html`)

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
      <a href="{{ url_for('ticket_list', project_id=project_id) }}">К списку заявок</a>
    </p>
  </section>

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
              <span class="comment-date">{{ c.created_at[:19] if c.created_at else '' }}</span>
            </div>
            <pre class="comment-body">{{ c.body }}</pre>
          </li>
        {% endfor %}
      </ul>
    {% endif %}

    {% if can_comment %}
      <hr />
      <h4>Добавить комментарий</h4>
      <form method="post" action="{{ url_for('comment_create', project_id=project_id, ticket_id=ticket.id) }}" class="form">
        <div class="form-group">
          <label for="body">Текст комментария</label>
          <textarea id="body" name="body" rows="3" required></textarea>
        </div>
        <button type="submit" class="btn btn-primary">Отправить</button>
      </form>
    {% else %}
      <p class="muted" style="margin-top: 8px;">Чтобы оставить комментарий, нужно быть участником проекта.</p>
    {% endif %}
  </section>

  <section class="card" style="margin-top: 16px;">
    <h3>История</h3>
    {% if not history %}
      <p class="muted">Для этой заявки пока нет записей в истории.</p>
    {% else %}
      <ul class="history-list">
        {% for h in history %}
          <li class="history-item">
            <div class="history-meta">
              <span class="history-date">{{ h.created_at[:19] if h.created_at else '' }}</span>
              <span class="history-user">{{ h.author_username or 'System' }}</span>
            </div>
            <div class="history-description">
              {% if h.action_type == 'created' %}
                Заявка создана.
              {% elif h.action_type == 'comment_added' %}
                Добавлен комментарий.
              {% elif h.action_type == 'status_changed' %}
                Статус заявки изменён.
              {% else %}
                Действие: {{ h.action_type }}.
              {% endif %}
            </div>
          </li>
        {% endfor %}
      </ul>
    {% endif %}
  </section>
{% endblock %}
```

---

## 2) История `comment_added` не записывается

### Что сломано

В `views_tickets.py` в `comment_create_view` есть ранний `return`, а блок `log_issue_event(..., "comment_added", ...)` стоит ниже и не выполняется.

### Что сделать

Замени функцию `comment_create_view` целиком:

```python
def comment_create_view(project_id: int, ticket_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    can_view, _ = can_view_ticket(ticket_id)
    if not can_view:
        abort(404)

    project = get_project(project_id)
    if project is None or project["is_archived"]:
        abort(403)

    body = (request.form.get("body") or "").strip()
    if not body:
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

    log_issue_event(
        ticket_id,
        "comment_added",
        {
            "project_id": project_id,
            "snippet": body[:100],
        },
    )

    return redirect(url_for("ticket_detail", project_id=project_id, ticket_id=ticket_id))
```

---

## 3) Неправильные ссылки на регистрацию

### Что сломано

Для открытия страницы регистрации используется `url_for('register')`, но это POST-роут.

### Что сделать

Исправь ссылки у **КНОПОК** (или у **ССЫЛОК**):

- В `templates/base.html`:  
  `{{ url_for('register') }}` -> `{{ url_for('register_form') }}`
- В `templates/login.html`:  
  `{{ url_for('register') }}` -> `{{ url_for('register_form') }}`

Важно: в самой **ФОРМЕ** `<form method="post" ...>` для отправки регистрации должен остаться `url_for('register')`!!!.

---

## 4) Нет проверки участника в `project_detail_view`

### Что сломано

Страница проекта может показываться залогиненному не-участнику.

### Что сделать

В `views_projects.py`, внутри функции `project_detail_view`, после загрузки проекта добавь проверку:

```python
project = get_project(project_id)
if project is None:
    abort(404)

if not can_view_project(project_id):
    abort(403)
```

---

## 5) Read-only для архивного проекта не доведен до конца

### Что сломано

- Не все POST-действия блокируются при `project.is_archived == 1`.
- В `comment_create_view` проверка архива была в мертвом коде (после `return`).
- Добавление/удаление участников сейчас можно выполнить даже для архивного проекта.

### Что сделать

#### 5.1 Блокировать создание заявки в архиве (`views_tickets.py`)

В функциях `ticket_new_view` и `ticket_create_view` почти в самом верху, сразу после проверки логина и до любой бизнес-логики (прав доступа, чтения формы, INSERT и т.д.), добавь:

```python
project = get_project(project_id)
if project is None or project["is_archived"]:
    abort(403)
```

Практически так:
- для ticket_new_view: после if not is_logged_in(): ...
- для ticket_create_view: после if not is_logged_in(): ...

#### 5.2 Блокировать изменение участников в архиве (`views_projects.py`)

В функциях `project_member_add_view` и `project_member_remove_view` после строчки `project = get_project(project_id)` добавить:

```python
if project is None:
    abort(404)

if project["is_archived"]:
    abort(403)
```

---

## 6) Кнопка "Восстановить проект" отправляет не туда

### Что сломано

В `templates/project_detail.html` текст кнопки "Восстановить проект", но `action` формы ведет на `project_archive`.

### Что сделать

Используй разные формы для archive/restore. Найди в `templates/project_detail.html` блок с формой, где `action="{{ url_for('project_archive', ...) }}"` и внутри `if not project.is_archived ... else ....:`. Удали этот старый блок и вставь новый: 

```html
{% if project_role == 'owner' %}
  {% if not project.is_archived %}
    <form method="post" action="{{ url_for('project_archive', project_id=project.id) }}" class="inline-form">
      <button
        type="submit"
        class="btn btn-sm"
        onclick="return confirm('Отправить проект в архив? Заявки и комментарии станут только для чтения.')"
      >
        Архивировать проект
      </button>
    </form>
  {% else %}
    <form method="post" action="{{ url_for('project_restore', project_id=project.id) }}" class="inline-form">
      <button type="submit" class="btn btn-sm">Восстановить проект</button>
    </form>
  {% endif %}

  {% if project.is_archived %}
    <form
      method="post"
      action="{{ url_for('project_delete', project_id=project.id) }}"
      class="inline-form"
      style="margin-left: 8px;"
    >
      <button
        type="submit"
        class="btn btn-sm btn-danger"
        onclick="return confirm('Удалить проект и все связанные данные безвозвратно?')"
      >
        Удалить проект
      </button>
    </form>
  {% endif %}
{% endif %}
```

---

## 8) Убрать `master/master`, читать admin из `.env`

### 8.1 Создай `.env` в корне проекта

```env
ADMIN_USERNAME=master
ADMIN_PASSWORD=change_me_123
```

### 8.2 Подключи загрузку `.env` (обязательно)

Установи пакет:

```bash
pip install python-dotenv
```

Добавь в `app.py` в самый верх (до запуска приложения и до вызова `ensure_master()`):

```python
from dotenv import load_dotenv
load_dotenv()
```

Почему это важно:

- `os.getenv(...)` читает переменные окружения;
- файл `.env` сам по себе не загружается;
- `load_dotenv()` переносит значения из `.env` в окружение.

### 8.3 Обнови `auth_utils.py`

Добавь импорт:

```python
import os
```

Замени `ensure_master()`:

```python
def ensure_master():
    """
    Убедиться, что в системе есть хотя бы один администратор.
    Если нет — создать из ADMIN_USERNAME / ADMIN_PASSWORD.
    """
    conn = get_conn()
    row = conn.execute(
        "SELECT id FROM users WHERE role = 'admin' AND archived_at IS NULL LIMIT 1"
    ).fetchone()
    conn.close()

    if row is not None:
        return

    username = os.getenv("ADMIN_USERNAME", "").strip()
    password = os.getenv("ADMIN_PASSWORD", "").strip()

    if not username or not password:
        raise RuntimeError(
            "Не заданы ADMIN_USERNAME / ADMIN_PASSWORD в .env для создания первого администратора."
        )

    create_user(username, password, "admin")
```

После этого `ensure_master()` будет читать `ADMIN_USERNAME` и `ADMIN_PASSWORD` из `.env` через `os.getenv(...)`.

---

## 9) Быстрая проверка после всех правок

1. Открой карточку заявки: комментарии и история видны.
2. Добавь комментарий: он появился и в комментариях, и в истории (`comment_added`).
3. Открой `/projects/<id>` не-участником: доступ запрещен (403/404).
4. Архивируй проект:
   - нельзя создать заявку;
   - нельзя добавить комментарий;
   - нельзя менять участников.
5. Нажми "Восстановить проект": проект восстанавливается (роут `project_restore`).
6. Ссылки "Регистрация" открывают страницу регистрации.

---

## 10) Если хочешь минимальный порядок коммитов

1. `fix(ticket-detail): render comments and history inside content block`
2. `fix(tickets): log comment_added and remove unreachable code`
3. `fix(authz): guard project detail and archive readonly writes`
4. `fix(project-ui): correct archive/restore form actions`
5. `chore(cleanup): remove static/app python duplicates`
6. `chore(auth): load admin bootstrap credentials from env`

