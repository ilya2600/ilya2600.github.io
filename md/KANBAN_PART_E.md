{% raw %}
# Часть E: Доски — первая предметная сущность Kanban

В Частях A–D вы подготовили общий каркас приложения:

- сервер Flask с базовым шаблоном и стилями;
- таблицу `users`, регистрацию, вход/выход, мастер‑аккаунт;
- флаг «регистрация открыта/закрыта»;
- минимальный админ‑раздел для управления пользователями.

Теперь для проекта Kanban мы добавим **первую предметную сущность** — **доску** (`board`):

- разберём таблицу `boards` в базе данных;
- напишем функции‑хелперы видимости и прав;
- сделаем список досок, форму создания и страницу просмотра одной доски.

Колонки (`columns`), карточки (`cards`), участники (`collaborators`) и история действий останутся на следующие части.

---

## E1. Таблица `boards` в `init_db`

Сначала убедимся, что в базе есть таблица для досок.

### 1. Смысл полей таблицы `boards`

Нам нужна таблица `boards`, которая хранит:

- `id` — идентификатор доски;
- `owner_id` — владелец доски (ссылка на `users.id`);
- `title` — название доски;
- `description` — краткое текстовое описание;
- `is_archived` — флаг архивации (0 — активна, 1 — в архиве);
- `created_at` — дата/время создания доски.

### 2. Добавляем создание таблицы в `init_db()`

Откройте `app.py` и найдите функцию `init_db()`. Внутри неё, после создания таблиц `settings` и `users`, добавьте создание таблицы `boards`:

```python
def init_db():
    ...
    conn.execute("""
        CREATE TABLE IF NOT EXISTS users ...""")

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

    conn.commit()
    conn.close()
```

Проверьте, что структура `boards` совпадает с описанием выше.

---

## E2. Хелперы видимости и прав для досок

Доску должен видеть только тот, кто имеет к ней доступ. Для первой версии Kanban‑проекта достаточно правила:

- владелец (`owner_id`) видит и редактирует доску;
- позже мы добавим приглашённых участников с ролями `viewer` / `editor`.

### 1. Функция `get_board_role(board_id)`

Добавьте в `app.py` (рядом с другими хелперами авторизации) функцию, которая возвращает роль текущего пользователя относительно доски:

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

### 2. Функции `can_view_board`, `can_edit_board`, `is_board_owner`

На основе `get_board_role` добавим три удобных хелпера:

```python
def can_view_board(board_id):
    """Можно ли просматривать эту доску."""
    return get_board_role(board_id) is not None


def can_edit_board(board_id):
    """Можно ли редактировать доску (менять название, добавлять колонки и карточки)."""
    role = get_board_role(board_id)
    return role == "owner"


def is_board_owner(board_id):
    """Является ли текущий пользователь владельцем доски."""
    return get_board_role(board_id) == "owner"
```

Пока эти функции используют только владельца доски. В следующих частях их можно будет расширить под совместную работу.

---

## E3. Список досок на дашборде

Сделаем так, чтобы после входа пользователь видел список своих досок. Для простоты:

- создадим маршрут `GET /dashboard`, который:
  - требует авторизации;
  - выбирает все активные доски (не в архиве), где текущий пользователь — владелец;
  - отдаёт их в шаблон `dashboard.html`.

### 1. Маршрут `GET /dashboard`

Добавьте в `app.py`:

```python
@app.get("/dashboard")
def dashboard():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    conn = get_conn()
    boards = conn.execute(
        """
        SELECT id, title, description, is_archived, created_at
        FROM boards
        WHERE owner_id = ? AND is_archived = 0
        ORDER BY created_at DESC
        """,
        (user["id"],),
    ).fetchall()
    conn.close()

    return render_template("dashboard.html", user=user, boards=boards)
```

### 2. Шаблон `dashboard.html` со списком досок

Создайте файл `templates/dashboard.html` (если ещё не создан) с простым списком досок:

```html
{% extends "base.html" %}

{% block title %}Дашборд{% endblock %}

{% block content %}
  <section class="card">
    <h1>Ваши доски</h1>

    <p>
      <a href="{{ url_for('board_new') }}" class="btn btn-primary">Новая доска</a>
    </p>

    {% if not boards %}
      <p class="muted">У вас пока нет ни одной доски.</p>
    {% else %}
      <table class="table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Название</th>
            <th>Описание</th>
            <th>Создана</th>
          </tr>
        </thead>
        <tbody>
          {% for b in boards %}
            <tr>
              <td><a href="{{ url_for('board_view', board_id=b.id) }}">{{ b.id }}</a></td>
              <td><a href="{{ url_for('board_view', board_id=b.id) }}">{{ b.title }}</a></td>
              <td>{{ b.description }}</td>
              <td>{{ b.created_at[:10] if b.created_at else '' }}</td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    {% endif %}
  </section>
{% endblock %}
```

Главное — ссылка на просмотр доски (`board_view`) и на создание новой (`board_new`), которые мы сейчас добавим.

---

## E4. Создание новой доски

Теперь добавим форму и маршруты для создания доски.

### 1. Маршруты `GET /boards/new` и `POST /boards/new`

В `app.py` добавьте:

```python
@app.get("/boards/new")
def board_new():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    return render_template("board_new.html", error=None)


@app.post("/boards/new")
def board_create():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    title = (request.form.get("title") or "").strip()
    description = (request.form.get("description") or "").strip()

    if not title:
        flash("Введите название доски.")
        return render_template("board_new.html", error="Введите название доски.")

    user = current_user()

    conn = get_conn()
    conn.execute(
        "INSERT INTO boards (owner_id, title, description) VALUES (?, ?, ?)",
        (user["id"], title, description or ""),
    )
    conn.commit()
    conn.close()

    return redirect(url_for("dashboard"))
```

### 2. Шаблон `board_new.html`

Создайте файл `templates/board_new.html`:

```html
{% extends "base.html" %}

{% block title %}Новая доска{% endblock %}

{% block content %}
  <section class="card">
    <h1>Новая доска</h1>

    {% if error %}
      <p class="bad">{{ error }}</p>
    {% endif %}

    <form method="post" action="{{ url_for('board_create') }}" class="form">
      <div class="form-group">
        <label for="title">Название</label>
        <input type="text" id="title" name="title" required />
      </div>

      <div class="form-group">
        <label for="description">Описание</label>
        <textarea id="description" name="description" rows="4"></textarea>
      </div>

      <button type="submit" class="btn btn-primary">Создать доску</button>
    </form>

    <p style="margin-top: 16px;">
      <a href="{{ url_for('dashboard') }}">К дашборду</a>
    </p>
  </section>
{% endblock %}
```

После этого на дашборде по кнопке «Новая доска» можно будет создать первую доску и вернуться к списку.

---

## E5. Просмотр одной доски

Сделаем страницу, на которой можно подробно посмотреть одну доску. Пока без колонок и карточек — только заголовок, описание и общая информация.

### 1. Маршрут `GET /boards/<int:board_id>`

Добавьте в `app.py`:

```python
@app.get("/boards/<int:board_id>")
def board_view(board_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    if not can_view_board(board_id):
        abort(404)

    conn = get_conn()
    board = conn.execute("""
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
    """, (board_id,)).fetchone()
    conn.close()

    if board is None:
        abort(404)

    return render_template(
        "board.html",
        board=board,
        is_owner=is_board_owner(board_id),
    )
```

### 2. Шаблон `board.html`

Создайте файл `templates/board.html`:

```html
{% extends "base.html" %}

{% block title %}Доска: {{ board.title }}{% endblock %}

{% block content %}
  <section class="card">
    <h1>{{ board.title }}</h1>

    <p class="muted">
      Владелец: {{ board.owner_username }}<br />
      Создана: {{ board.created_at }}
      {% if board.is_archived %}
        <br /><span class="badge badge-deleted">В архиве</span>
      {% endif %}
    </p>

    {% if board.description %}
      <h2>Описание</h2>
      <p>{{ board.description }}</p>
    {% endif %}

    <h2>Колонки и карточки</h2>
    <p class="muted">
      Здесь позже появятся колонки и карточки Kanban‑доски.
    </p>

    <p style="margin-top: 16px;">
      <a href="{{ url_for('dashboard') }}">К дашборду</a>
    </p>
  </section>
{% endblock %}
```

На этом шаге мы только выводим общую информацию о доске и оставляем «заглушку» под будущие колонки и карточки.

---

## Итог Части E

После выполнения этой части у вас есть:

- таблица `boards` с полями владельца, названия, описания, флагом архива и датой создания;
- хелперы прав доступа к доскам: `get_board_role`, `can_view_board`, `can_edit_board`, `is_board_owner`;
- дашборд `/dashboard` со списком досок текущего пользователя;
- форма создания доски `/boards/new` и переход на неё;
- страница просмотра одной доски `/boards/<id>`.

Это первый шаг предметной логики Kanban‑проекта: пользователь может заводить свои доски и открывать их для просмотра. В следующих частях вы добавите:

- колонки и карточки внутри доски;
- перемещение карточек и WIP‑лимиты;
- приглашение других пользователей к совместной работе с разными ролями.
{% endraw %}

