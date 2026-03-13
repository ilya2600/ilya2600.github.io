{% raw %}
# Часть F: Страницы — первая предметная сущность вики

В Частях A–E вы разнесли приложение по модулям:

- `app.py` — создание Flask‑приложения и тонкие маршруты;
- `db.py` — работа с базой (`init_db`, `get_conn` и т.п.);
- `auth_utils.py` — пользователи и права (`current_user`, `is_admin`, `is_logged_in`, `get_registration_open`);
- `views_auth.py` — логика входа, регистрации и дашборда;
- `views_admin.py` — логика админ‑страниц;
- шаблоны в `templates/`.

В этой части вы добавляете **первую предметную сущность вики** — **страницу** (`page`):

- обновлённую таблицу `users` с ролями вики и таблицу `pages` в `db.py`;
- функцию `is_author_or_above()` в `auth_utils.py`;
- модуль `views_pages.py` с хелперами для slug, списком страниц, созданием и просмотром по slug;
- маршруты `/pages`, `/pages/new`, `/pages/<slug>` в `app.py`;
- шаблоны списка, формы создания и просмотра страницы.

Ревизии, история правок и блокировки страниц остаются на следующие части.

---

## F1. Схема БД: таблица `users` (роли вики) и таблица `pages` в `db.py`

Для вики нужны роли `author`, `editor`, `admin` и одна таблица страниц. Содержимое страницы храним в самой таблице `pages` (поле `content`); таблицу ревизий введём позже.

### Шаг F1.1. Обновлённая таблица `users` и таблица `pages`

Откройте `db.py` и найдите функцию `init_db()`. Замените создание таблицы `users` на вариант с ролями вики и сразу после неё добавьте таблицу `pages`:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT NOT NULL UNIQUE,
            password TEXT NOT NULL,
            role TEXT NOT NULL CHECK (role IN ('author', 'editor', 'admin')),
            archived_at TEXT,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        )
    """)

    conn.execute("""
        CREATE TABLE IF NOT EXISTS pages (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL UNIQUE,
            slug TEXT NOT NULL UNIQUE,
            content TEXT NOT NULL DEFAULT '',
            author_id INTEGER,
            is_locked INTEGER NOT NULL DEFAULT 0,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            FOREIGN KEY (author_id) REFERENCES users(id)
        )
    """)
```

Поле `password` в `users` у вас может называться `password_hash` и заполняться хешем — оставьте как в текущем проекте. Важно: **роль** задаётся как `CHECK (role IN ('author', 'editor', 'admin'))`.

Таблица `pages`: `id`, уникальные `title` и `slug`, `content` (тело страницы), `author_id` (кто создал), `is_locked`, `created_at`.

### Шаг F1.2. Удаление старой базы после смены схемы

**Важно:** `CREATE TABLE IF NOT EXISTS` не меняет уже существующую таблицу. Если у вас уже есть файл `database.db` со старой таблицей `users` (например, с `role IN ('user', 'admin')`), после смены схемы в коде удалите файл базы:

- удалите файл `database.db` в корне проекта (или по пути из `DB_PATH` в `db.py`);
- при следующем запуске приложения `init_db()` создаст таблицы заново с новыми ролями.

Иначе приложение может падать из‑за недопустимых значений ролей в старых данных.

### Шаг F1.3. Вызов `insert_test_user()` в `app.py`

В универсальном проекте функция `insert_test_user()` в `db.py` создаёт тестового пользователя с ролью `'user'`, которая в вики больше не допускается. При запуске приложения в блоке `if __name__ == "__main__":` эта функция вызывается и приведёт к нарушению ограничения CHECK.

**В `app.py` в блоке запуска закомментируйте или удалите вызов `insert_test_user()`**, чтобы при старте не создавался пользователь с недопустимой ролью:

```python
if __name__ == "__main__":
    init_db()
    ensure_master()
    # insert_test_user()  # отключено: создаёт роль 'user', в вики допустимы только author, editor, admin
    app.run(debug=True)
```

Пользователей для проверки после смены схемы можно создавать через админ‑страницу или форму регистрации (с допустимыми ролями).

---

## F2. Функция `is_author_or_above` в `auth_utils.py`

Создавать и править страницы могут только пользователи с ролью author, editor или admin.

### Шаг F2.1. Добавляем функцию

Откройте `auth_utils.py` и рядом с `is_admin()` добавьте:

```python
def is_author_or_above():
    """Может ли текущий пользователь создавать и править страницы (роли author, editor, admin)."""
    u = current_user()
    return u is not None and u["role"] in ("author", "editor", "admin")
```

В вики «мастер» — роль `admin`; для админ‑ссылок в шаблонах по‑прежнему используется `is_admin()` из контекста.

### Шаг F2.2. Регистрация: роль `author` в `views_auth.py`

Самостоятельная регистрация должна создавать пользователей с одной из допустимых ролей вики. По умолчанию для новых пользователей задаём роль **author**.

В файле `views_auth.py` в функции `register_view()` замените вставку в `users`, в которой используется роль `'user'`, на роль `'author'`:

```python
    conn.execute(
        "INSERT INTO users (username, password, role) VALUES (?, ?, 'author')",
        (username, generate_password_hash(password)),
    )
```

Если у вас в таблице столбец называется `password_hash`, подставьте его в запрос и передавайте уже посчитанный хеш, как в текущем коде.

---

## F3. Модуль `views_pages.py` — хелперы и логика страниц

Вся логика страниц вики выносится в отдельный модуль.

### Шаг F3.1. Создаём файл `views_pages.py`

В той же папке, где лежат `app.py`, `views_auth.py`, `views_admin.py`, создайте файл `views_pages.py`.

### Шаг F3.2. Импорты

В начало `views_pages.py` вставьте:

```python
import re
from flask import render_template, redirect, url_for, request, abort, flash

from db import get_conn
from auth_utils import is_logged_in, current_user, is_author_or_above
```

### Шаг F3.3. Хелперы для slug и страниц

Добавьте в `views_pages.py`:

```python
def slugify(title: str) -> str:
    """Преобразовать заголовок в slug для URL."""
    s = title.lower().strip()
    s = re.sub(r"[^\w\s-]", "", s)
    s = re.sub(r"[-\s]+", "-", s)
    return s or "page"


def ensure_unique_slug(conn, slug: str, exclude_page_id=None) -> str:
    """Гарантировать уникальность slug. Если занят — добавляет -1, -2, ..."""
    base = slug
    n = 0
    while True:
        candidate = f"{base}-{n}" if n else base
        row = conn.execute(
            "SELECT id FROM pages WHERE slug = ?",
            (candidate,),
        ).fetchone()
        if row is None or (exclude_page_id is not None and row["id"] == exclude_page_id):
            return candidate
        n += 1


def get_page_by_slug(slug: str):
    """Найти страницу по slug с именем автора."""
    conn = get_conn()
    page = conn.execute(
        """
        SELECT
            p.id,
            p.title,
            p.slug,
            p.content,
            p.author_id,
            p.is_locked,
            p.created_at,
            u.username AS author_username
        FROM pages p
        LEFT JOIN users u ON p.author_id = u.id
        WHERE p.slug = ?
        """,
        (slug,),
    ).fetchone()
    conn.close()
    return page
```

### Шаг F3.4. View: список страниц

```python
def page_list_view():
    conn = get_conn()
    pages = conn.execute(
        "SELECT id, title, slug, created_at FROM pages ORDER BY title"
    ).fetchall()
    conn.close()
    return render_template("page_list.html", pages=pages)
```

### Шаг F3.5. View: форма создания и обработка

Доступ только у залогиненных с ролью author/editor/admin:

```python
def page_new_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_author_or_above():
        abort(403)
    return render_template("page_new.html", error=None)


def page_create_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_author_or_above():
        abort(403)

    title = (request.form.get("title") or "").strip()
    content = request.form.get("content") or ""

    if not title:
        return render_template("page_new.html", error="Введите заголовок страницы.")

    conn = get_conn()
    existing = conn.execute(
        "SELECT id FROM pages WHERE title = ?",
        (title,),
    ).fetchone()
    if existing is not None:
        conn.close()
        return render_template("page_new.html", error="Страница с таким заголовком уже существует.")

    slug = slugify(title)
    slug = ensure_unique_slug(conn, slug)
    user = current_user()

    conn.execute(
        "INSERT INTO pages (title, slug, content, author_id) VALUES (?, ?, ?, ?)",
        (title, slug, content, user["id"]),
    )
    conn.commit()
    conn.close()

    flash("Страница создана.")
    return redirect(url_for("page_view", slug=slug))
```

### Шаг F3.6. View: просмотр одной страницы по slug

```python
def page_view_view(slug: str):
    page = get_page_by_slug(slug)
    if page is None:
        abort(404)
    return render_template("page_view.html", page=page)
```

---

## F4. Маршруты страниц вики в `app.py`

### Шаг F4.1. Импорт view‑функций

В `app.py` в блок импортов из view‑модулей добавьте:

```python
from views_pages import (
    page_list_view,
    page_new_view,
    page_create_view,
    page_view_view,
)
```

### Шаг F4.2. Объявление маршрутов

Ниже существующих маршрутов добавьте:

```python
@app.get("/pages")
def page_list():
    return page_list_view()


@app.get("/pages/new")
def page_new():
    return page_new_view()


@app.post("/pages/new")
def page_create():
    return page_create_view()


@app.get("/pages/<slug>")
def page_view(slug):
    return page_view_view(slug)
```

---

## F5. Шаблон списка страниц `page_list.html`

Создайте файл `templates/page_list.html`:

```html
{% extends "base.html" %}

{% block title %}Страницы · Вики{% endblock %}

{% block content %}
  <section class="card">
    <h1>Страницы</h1>

    <p>
      <a href="{{ url_for('page_new') }}" class="btn btn-primary">Создать страницу</a>
    </p>

    {% if not pages %}
      <p class="muted">Пока нет ни одной страницы.</p>
    {% else %}
      <table class="table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Заголовок</th>
            <th>Создана</th>
          </tr>
        </thead>
        <tbody>
          {% for page in pages %}
            <tr>
              <td>{{ page.id }}</td>
              <td>
                <a href="{{ url_for('page_view', slug=page.slug) }}">{{ page.title }}</a>
              </td>
              <td>{{ page.created_at[:10] if page.created_at else '' }}</td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    {% endif %}
  </section>
{% endblock %}
```

---

## F6. Шаблон создания страницы `page_new.html`

Создайте файл `templates/page_new.html`:

```html
{% extends "base.html" %}

{% block title %}Новая страница · Вики{% endblock %}

{% block content %}
  <section class="card">
    <h1>Новая страница</h1>

    {% if error %}
      <p class="bad">{{ error }}</p>
    {% endif %}

    <form method="post" action="{{ url_for('page_create') }}" class="form">
      <div class="form-group">
        <label for="title">Заголовок</label>
        <input type="text" id="title" name="title" required />
      </div>

      <div class="form-group">
        <label for="content">Содержимое</label>
        <textarea id="content" name="content" rows="10"></textarea>
      </div>

      <button type="submit" class="btn btn-primary">Создать страницу</button>
    </form>

    <p style="margin-top: 16px;">
      <a href="{{ url_for('page_list') }}">К списку страниц</a>
    </p>
  </section>
{% endblock %}
```

---

## F7. Шаблон просмотра страницы `page_view.html`

Создайте файл `templates/page_view.html`:

```html
{% extends "base.html" %}

{% block title %}{{ page.title }} · Вики{% endblock %}

{% block content %}
  <section class="card">
    <h1>{{ page.title }}</h1>

    <p class="muted">
      {% if page.author_username %}
        Автор: {{ page.author_username }}<br />
      {% endif %}
      Создана: {{ page.created_at }}
    </p>

    <div class="wiki-content">
      <pre>{{ page.content }}</pre>
    </div>

    <p style="margin-top: 16px;">
      <a href="{{ url_for('page_list') }}">К списку страниц</a>
    </p>
  </section>
{% endblock %}
```

Переменная `page` передаётся из `page_view_view()` и содержит поля страницы, в том числе `content` и `author_username` (из `get_page_by_slug`).

---

## F8. Навигация в `base.html` для вики

В текущем проекте в шаблоны передаются переменная `registration_open` и функция `is_admin()`. В примере ниже для ссылки на регистрацию используется **`url_for('register')`** — имя маршрута должно совпадать с тем, как объявлен обработчик GET `/register` в `app.py` (если у вас он называется `register_form`, подставьте `url_for('register_form')`).

Пример блока навигации с учётом вики:

```html
<nav>
  <a href="{{ url_for('home') }}">Главная</a>
  <a href="{{ url_for('page_list') }}">Страницы</a>

  {% if current_user() %}
    <a href="{{ url_for('dashboard') }}">Дашборд</a>
    {% if is_admin() %}
      <a href="{{ url_for('admin_settings') }}">Настройки</a>
      <a href="{{ url_for('admin_users') }}">Пользователи</a>
    {% endif %}
    <span style="float: right;">
      {{ current_user().username }}
      (<a href="{{ url_for('logout') }}">выйти</a>)
    </span>
  {% else %}
    <a href="{{ url_for('login_form') }}">Вход</a>
    {% if registration_open %}
      <a href="{{ url_for('register') }}">Регистрация</a>
    {% endif %}
  {% endif %}
</nav>
```

Имена маршрутов в шаблоне должны совпадать с объявлениями в `app.py`.

---

## F9. Администратор: создание пользователя с выбором роли в `views_admin.py`

В универсальном варианте в `admin_user_create_view()` пользователь создаётся с жёстко заданной ролью `'user'`, которая в вики не допускается. Нужно брать роль из формы и проверять её по списку допустимых ролей вики.

**В файле `views_admin.py` в функции `admin_user_create_view()` замените блок, в котором вызывается `create_user`.** Считывайте поле `role` из формы, проверяйте его по кортежу `('author', 'editor', 'admin')`, при недопустимом значении подставляйте `'author'`, затем вызывайте `create_user(username, password, role)`:

```python
    username = (request.form.get("username") or "").strip()
    password = request.form.get("password") or ""
    role = (request.form.get("role") or "author").strip().lower()

    if not username or not password:
        flash("Введите логин и пароль.")
        return redirect(url_for("admin_users"))

    if len(username) < 2:
        flash("Логин слишком короткий.")
        return redirect(url_for("admin_users"))

    allowed_roles = ("author", "editor", "admin")
    if role not in allowed_roles:
        role = "author"

    try:
        create_user(username, password, role)
        flash(f"Пользователь {username} создан.")
    except sqlite3.IntegrityError:
        flash("Такой логин уже занят.")

    return redirect(url_for("admin_users"))
```

---

## F10. Администратор: выпадающий список ролей в `templates/admin_users.html`

В форме создания пользователя на админ‑странице администратор должен выбирать роль (author, editor, admin). Иначе в форму не передаётся значение `role`, и логика из F9 будет подставлять `'author'` по умолчанию — но явный выбор роли удобнее.

**В `templates/admin_users.html` внутри формы, которая отправляет данные в `admin_user_create`, добавьте выпадающий список для выбора роли.** Разместите его после поля пароля и перед кнопкой отправки:

```html
        <div class="form-group">
            <label for="new_role">Роль</label>
            <select id="new_role" name="role">
                <option value="author">Автор (author)</option>
                <option value="editor">Редактор (editor)</option>
                <option value="admin">Мастер (admin)</option>
            </select>
        </div>
```

Имя поля `name="role"` должно совпадать с тем, что читается в `admin_user_create_view()` как `request.form.get("role")`.

---

## Итог Части F (Вики)

После выполнения этой части в проекте есть:

- **`db.py`**: таблица `users` с ролями `author`, `editor`, `admin`; таблица `pages` (title, slug, content, author_id, is_locked, created_at). После смены схемы удалён старый `database.db`; вызов `insert_test_user()` в `app.py` закомментирован.
- **`auth_utils.py`**: функция `is_author_or_above()`.
- **`views_auth.py`**: в `register_view()` при регистрации создаётся пользователь с ролью `'author'`.
- **`views_admin.py`**: в `admin_user_create_view()` роль берётся из формы, проверяется по списку `('author', 'editor', 'admin')`, по умолчанию `'author'`; вызов `create_user(username, password, role)`.
- **`views_pages.py`**: хелперы `slugify`, `ensure_unique_slug`, `get_page_by_slug`; view‑функции `page_list_view()`, `page_new_view()`, `page_create_view()`, `page_view_view()`.
- **`app.py`**: маршруты `/pages`, `/pages/new` (GET и POST), `/pages/<slug>`.
- **Шаблоны**: `page_list.html`, `page_new.html`, `page_view.html`; в `admin_users.html` — выпадающий список ролей (author, editor, admin); навигация в `base.html` с `registration_open`, `is_admin()` и нужным именем маршрута регистрации.

Это первый шаг предметной логики вики: одна сущность «страница», список страниц, создание и просмотр по slug; все пути создания пользователей (регистрация и админ) используют только допустимые роли. В следующих частях — ревизии, история правок, блокировки.
{% endraw %}
