{% raw %}
# Часть F: Доски Kanban — первая предметная сущность

В Частях A–E вы подготовили общий каркас приложения и описали первую предметную сущность Kanban‑проекта — **доску** (`board`):

- базовая аутентификация и роли пользователей;
- структура базы с таблицами `settings` и `users`;
- идея таблицы `boards` для хранения досок;
- дашборд со списком досок пользователя;
- создание новой доски;
- страница просмотра одной доски.

В этой части вы **пошагово реализуете именно эти возможности**:

- добавите таблицу `boards` в инициализацию базы;
- оформите функции прав доступа к доскам;
- создадите модуль с логикой досок;
- подключите маршруты в `app.py`;
- создадите шаблоны `dashboard.html`, `board_new.html` и `board.html`.

Никаких колонок, карточек и приглашённых участников в этой части ещё нет — мы **строго ограничиваемся тем, что описано в Части E для Kanban**.

---

## F1. Таблица `boards` в инициализации базы

Сначала убедимся, что в базе данных есть таблица `boards`, соответствующая описанию из Части E.

### Шаг F1.1. Находим и открываем инициализацию базы

Откройте файл, в котором у вас сейчас создаются таблицы `settings` и `users` (после Части E это обычно либо `db.py` с функцией `init_db()`, либо та же логика в `app.py`).

Найдите функцию:

```python
def init_db():
    ...
```

Внутри неё уже должны создаваться таблицы настроек и пользователей через `conn.execute("""CREATE TABLE IF NOT EXISTS ...""")`.

### Шаг F1.2. Добавляем таблицу `boards`

Сразу **после** блока `CREATE TABLE IF NOT EXISTS users (...)` добавьте создание таблицы `boards`:

```python
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
```

Проверьте, что:

- поля совпадают с описанием из Части E;
- `owner_id` ссылается на `users(id)`;
- `is_archived` по умолчанию `0`;
- `created_at` заполняется автоматически через `strftime(...)`.

Функция `init_db()` в итоге должна:

- создавать `settings`;
- создавать `users`;
- создавать `boards`;
- вызывать `conn.commit()` и `conn.close()`.

Если часть кода уже была добавлена ранее по Части E, просто сравните и приведите всё к единому виду.

---

## F2. Хелперы прав доступа к доскам

Теперь реализуем функции, которые определяют, **какое отношение текущий пользователь имеет к конкретной доске**. Они будут основой для проверки прав во view‑функциях.

В Части E мы обсуждали функции:

- `get_board_role(board_id)`,
- `can_view_board(board_id)`,
- `can_edit_board(board_id)`,
- `is_board_owner(board_id)`.

Здесь мы оформим их в коде.

### Шаг F2.1. Выбираем место для хелперов

Откройте файл, где у вас уже лежат функции, связанные с пользователями и правами:

- если вы следовали Части E (общей) — это, скорее всего, `auth_utils.py`;

Для дальнейших частей удобнее хранить хелперы в модуле `auth_utils.py`. Ниже мы будем считать, что вы используете именно его. Если у вас всё ещё в `app.py` — код функций будет тем же, просто разместите его там.

### Шаг F2.2. Реализуем `get_board_role`

В `auth_utils.py` добавьте функцию:

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

Обратите внимание:

- если пользователь не вошёл — возвращаем `None`;
- если доска не найдена — тоже `None`;
- если текущий пользователь совпадает с `owner_id` — возвращаем `"owner"`;
- никаких других ролей мы пока не поддерживаем.

### Шаг F2.3. Функции `can_view_board`, `can_edit_board`, `is_board_owner`

Под `get_board_role` добавьте:

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

Эти функции будут использоваться во view‑логике, чтобы:

- решить, можно ли показывать доску (`can_view_board`);
- решить, можно ли будет в будущем редактировать её (`can_edit_board`, `is_board_owner`).

---

## F3. Модуль `views_boards.py` с логикой досок

Чтобы не перегружать `app.py`, всю логику, связанную с досками, удобно вынести в отдельный модуль.

В Части E мы уже наметили:

- дашборд `/dashboard` со списком досок;
- создание новой доски `/boards/new`;
- просмотр одной доски `/boards/<id>`.

Сейчас мы реализуем это как набор view‑функций без декораторов.

### Шаг F3.1. Создаём файл `views_boards.py`

В той же папке, где лежит `app.py`, создайте новый файл:

- `views_boards.py`

### Шаг F3.2. Импорты

В начало `views_boards.py` вставьте:

```python
from flask import render_template, redirect, url_for, request, abort

from db import get_conn
from auth_utils import is_logged_in, current_user, can_view_board, is_board_owner
```

Если ваши модули называются чуть иначе — поправьте импорты под свой проект, сохранив общую идею.

### Шаг F3.3. Хелпер `boards_for_user`

Сделаем маленькую функцию, которая возвращает **список активных досок текущего пользователя**:

```python
def boards_for_user():
    """Вернуть активные доски текущего пользователя."""
    user = current_user()
    if user is None:
        return []

    conn = get_conn()
    rows = conn.execute(
        """
        SELECT id, title, description, is_archived, created_at
        FROM boards
        WHERE owner_id = ? AND is_archived = 0
        ORDER BY created_at DESC
        """,
        (user["id"],),
    ).fetchall()
    conn.close()

    return rows
```

Это ровно та выборка, которую мы описывали в Части E для дашборда.

### Шаг F3.4. View‑функция дашборда

Теперь в `view` функции маршрута `/dashboard` (`dashboard_view()`) которая лежит в `views_auth.py`:

```python
def dashboard_view():
    if not is_logged_in():
        ...

    user = current_user()
    if user is None:
        ...
    
    boards = boards_for_user()
    return render_template(..., boards=boards)
```

и сверху файла добавьте:

```python
from views_boards import boards_for_user
```

Эта функция:
- подставляет в шаблон текущего пользователя и список его досок.

### Шаг F3.5. View‑функции создания доски

Добавим две функции в `views_boards.py` под форму и обработчик создания:

```python
def board_new_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    return render_template("board_new.html", error=None)


def board_create_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    title = (request.form.get("title") or "").strip()
    description = (request.form.get("description") or "").strip()

    if not title:
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

Поведение:

- доступ только для залогиненных пользователей;
- обязательное поле `title`;
- `description` можно оставить пустым;
- после успешного создания — редирект на дашборд.

### Шаг F3.6. View‑функция просмотра одной доски

Наконец, в `views_boards.py` реализуем страницу просмотра одной доски, используя `can_view_board` и `is_board_owner`:

```python
def board_view_view(board_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    if not can_view_board(board_id):
        abort(404)

    conn = get_conn()
    board = conn.execute(
        """
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
        """,
        (board_id,),
    ).fetchone()
    conn.close()

    if board is None:
        abort(404)

    return render_template(
        "board.html",
        board=board,
        is_owner=is_board_owner(board_id),
    )
```

Здесь:

- неавторизованных отправляем на страницу логина;
- если `can_view_board` возвращает `False` — отвечаем `404`, не раскрывая факт существования доски;
- данные владельца (`owner_username`) берём из `users`;
- в шаблон дополнительно передаём `is_owner`, чтобы позже можно было показывать дополнительные элементы управления только владельцу.

---

## F4. Подключаем маршруты досок в `app.py`

Теперь свяжем наш модуль `views_boards.py` с маршрутизатором Flask.

### Шаг F4.1. Импортируем view‑функции

Откройте `app.py` и рядом с импортами других view‑модулей добавьте:

```python
from views_boards import (
    board_new_view,
    board_create_view,
    board_view_view,
)
```

### Шаг F4.2. Объявляем маршруты

Ниже существующих маршрутов (например, после логина / админ‑части) добавьте «тонкие» функции, которые делегируют логику в `views_boards`:

```python
@app.get("/boards/new")
def board_new():
    return board_new_view()

@app.post("/boards/new")
def board_create():
    return board_create_view()

@app.get("/boards/<int:board_id>")
def board_view(board_id):
    return board_view_view(board_id)
```

Важно:

- в `app.py` **не должно быть дублирующей логики** выборки досок — всё уже есть в `views_boards.py`;
- `board_new` и `board_create` обрабатывают создание новой доски;
- `board_view` показывает одну доску по её `id`.

---

## F5. Шаблоны `dashboard.html`, `board_new.html`, `board.html`

Осталось оформить HTML‑шаблоны, соответствующие описанию из Части E.

Все шаблоны ниже предполагают, что у вас уже есть базовый шаблон `base.html` с блоками `title` и `content`, а также базовые стили.

### Шаг F5.1. Шаблон дашборда `dashboard.html`

В папке `templates` в `dashboard.html` вставьте в него:

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

Этот шаблон ожидает, что в контекст передан список `boards`, который мы как раз формируем в `dashboard_view`.

### Шаг F5.2. Шаблон создания доски `board_new.html`

В папке `templates` создайте файл:

- `board_new.html`

Содержимое:

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

Шаблон:

- показывает возможное сообщение об ошибке;
- содержит поля `title` и `description`;
- после отправки форма попадает на маршрут `board_create`.

### Шаг F5.3. Шаблон просмотра доски `board.html`

В папке `templates` создайте файл:

- `board.html`

Содержимое:

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

Пока это просто аккуратная страница с информацией о доске и заглушкой под будущие колонки и карточки.

---

## Итог Части F (Kanban)

После выполнения этой части у вас есть:

- таблица `boards` в базе данных с нужными полями;
- хелперы прав доступа к доскам: `get_board_role`, `can_view_board`, `can_edit_board`, `is_board_owner`;
- модуль `views_boards.py` с логикой:
  - `boards_for_user`,
  - `dashboard_view`,
  - `board_new_view`,
  - `board_create_view`,
  - `board_view_view`;
- маршруты в `app.py`, которые реализуют:
  - список досок `/dashboard`,
  - создание доски `/boards/new`,
  - просмотр одной доски `/boards/<id>`;
- три шаблона: `dashboard.html`, `board_new.html`, `board.html`.

Это полноценная первая часть предметной логики Kanban‑проекта: пользователь может создавать свои доски и открывать их для просмотра. В следующих частях вы добавите:

- колонки и карточки внутри доски;
- перемещение карточек и ограничения WIP;
- совместную работу нескольких пользователей на одной доске.
{% endraw %}

