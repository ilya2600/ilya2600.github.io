{% raw %}
# Часть E: Страницы и ревизии в вики

В Частях A–D вы:

- настроили базовое Flask‑приложение, шаблоны и стили;
- создали таблицу `users`, регистрацию, вход/выход и мастер‑аккаунт;
- добавили настройку «регистрация открыта/закрыта» и страницу `/admin/settings`;
- сделали админ‑страницу пользователей `/admin/users` с архивированием и удалением;
- вынесли общую шапку и навигацию в `base.html`.

Все эти шаги были **универсальными** и не зависели от темы проекта. В этой части мы начинаем предметную логику именно для вики:

- введём сущность **страницы** (`pages`);
- добавим **простые ревизии** (`revisions`);
- сделаем создание страницы, список страниц и просмотр одной страницы.

Статусы ревизий, модерация, откаты и блокировки добавим в следующих частях.

---

## E1. Таблица `pages` в `init_db`

Сначала создадим таблицу для хранения страниц вики.

### 1. Смысл полей таблицы `pages`

Нам нужно хранить:

- `id` — числовой идентификатор страницы;
- `title` — заголовок страницы (должен быть уникальным, чтобы не было двух страниц с одинаковым названием);
- `slug` — «человекочитаемый» идентификатор для URL (например, `python-basics` вместо `id`); тоже должен быть уникальным;
- `is_locked` — флаг блокировки:
  - `0` — страницу можно править авторам;
  - `1` — страницу могут править только редакторы/мастер (введём позже);
- `created_at` — дата/время создания страницы.

### 2. Добавляем `CREATE TABLE pages` в `init_db`

Откройте `app.py` и найдите функцию `init_db()`. Внутри неё, после создания `users` и при необходимости других таблиц, добавьте:

```python
def init_db():
    conn = get_conn()

    conn.execute("""
        CREATE TABLE IF NOT EXISTS settings (
            key TEXT PRIMARY KEY,
            value TEXT NOT NULL
        )
    """)
    conn.execute("INSERT OR IGNORE INTO settings (key, value) VALUES ('registration_open', '0')")

    conn.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT NOT NULL UNIQUE,
            password_hash TEXT NOT NULL,
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
            is_locked INTEGER NOT NULL DEFAULT 0,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        )
    """)

    conn.commit()
    conn.close()
```

> Если у вас в `init_db()` уже есть другие таблицы — просто впишите `pages` в подходящее место, следя за тем, чтобы `users` создавалась раньше (так как в следующих шагах мы будем ссылаться на пользователей из ревизий).

---

## E2. Таблица `revisions` — минимальный вариант

Полноценная вики хранит историю правок. Для этого нам нужна таблица `revisions`, где каждая строка — отдельная версия содержимого страницы.

### 1. Смысл полей таблицы `revisions`

Для упрощённой версии нам достаточно:

- `id` — идентификатор ревизии;
- `page_id` — ссылка на `pages.id`;
- `author_id` — пользователь, создавший эту версию (может быть `NULL`, если автор удалён);
- `content` — текст страницы в данной ревизии;
- `status` — строка со статусом ревизии:
  - на этом шаге можно использовать только `'approved'`, т.е. сразу публиковать;
- `created_at` — дата/время создания ревизии.

Более сложные поля (`reviewer_id`, `review_note`, `rollback_reason`, статусы `draft`/`pending`/`rejected`) мы добавим позже.

### 2. Добавляем `CREATE TABLE revisions` в `init_db`

В `init_db()` (сразу после `pages`) добавьте:

```python
def init_db():
    conn = get_conn()
    # ... settings и users ...

    conn.execute("""
        CREATE TABLE IF NOT EXISTS pages (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL UNIQUE,
            slug TEXT NOT NULL UNIQUE,
            is_locked INTEGER NOT NULL DEFAULT 0,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        )
    """)

    conn.execute("""
        CREATE TABLE IF NOT EXISTS revisions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            page_id INTEGER NOT NULL,
            author_id INTEGER,
            content TEXT NOT NULL,
            status TEXT NOT NULL CHECK (status IN ('approved')),
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            FOREIGN KEY (page_id) REFERENCES pages(id) ON DELETE CASCADE,
            FOREIGN KEY (author_id) REFERENCES users(id)
        )
    """)

    conn.commit()
    conn.close()
```

> Пока мы разрешаем только статус `'approved'`. Это означает, что каждая созданная ревизия сразу считается «опубликованной». В следующих частях мы расширим список статусов.

---

## E3. Хелперы для slug и страниц

Чтобы удобно работать со страницами в URL‑ах, нам нужны вспомогательные функции.

### 1. Функция `slugify(title)`

`slug` — это часть URL, которая:

- пишется латиницей и цифрами;
- не содержит пробелов и спецсимволов;
- получается из заголовка предсказуемым образом.

Добавьте в `app.py` (рядом с другими хелперами) функцию:

```python
import re


def slugify(title: str) -> str:
    """Преобразовать заголовок в человекочитаемый slug для URL."""
    s = title.lower().strip()
    # Удаляем всё, кроме букв, цифр, подчёркиваний, дефисов и пробелов
    s = re.sub(r"[^\w\s-]", "", s)
    # Заменяем последовательности пробелов/дефисов одним дефисом
    s = re.sub(r"[-\s]+", "-", s)
    return s or "page"
```

### 2. Функция `ensure_unique_slug(conn, slug, exclude_page_id=None)`

Нужно убедиться, что slug уникален. Если slug уже занят — добавляем суффикс `-1`, `-2` и т.д.

Добавьте:

```python
def ensure_unique_slug(conn, slug: str, exclude_page_id: int | None = None) -> str:
    """Гарантировать уникальность slug в таблице pages.

    Если slug уже занят, добавляет -1, -2, ... пока не найдёт свободный.
    exclude_page_id позволяет переиспользовать slug для редактируемой страницы.
    """
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
```

### 3. Функция `get_page_by_slug(slug)`

Чтобы находить страницу по URL, добавим:

```python
def get_page_by_slug(slug: str):
    conn = get_conn()
    page = conn.execute(
        "SELECT id, title, slug, is_locked, created_at FROM pages WHERE slug = ?",
        (slug,),
    ).fetchone()
    conn.close()
    return page
```

### 4. Функция `get_current_revision_id(conn, page_id)`

Для Part E достаточно брать **последнюю созданную ревизию** для страницы:

```python
def get_current_revision_id(conn, page_id: int) -> int | None:
    row = conn.execute(
        """
        SELECT id
        FROM revisions
        WHERE page_id = ? AND status = 'approved'
        ORDER BY created_at DESC
        LIMIT 1
        """,
        (page_id,),
    ).fetchone()
    return row["id"] if row else None
```

В следующих частях мы будем использовать эту функцию, чтобы показывать актуальную версию страницы.

---

## E4. Создание новой страницы `/pages/new`

Сделаем форму, позволяющую авторизованному пользователю создать страницу вики.

### 1. Доступ к созданию страницы

В упрощённом варианте можно разрешить создание страниц всем, кто вошёл (`is_logged_in()`), или только пользователям с ролями `author`, `editor`, `admin`. Для наглядности будем проверять роль.

Допустим, у вас уже есть функции:

- `is_master()` — проверяет `role == 'admin'`;
- `is_editor()` / `is_author_or_above()` — можно добавить позже.

Для этой части достаточно:

```python
def is_author_or_above():
    u = current_user()
    return u is not None and u["role"] in ("author", "editor", "admin")
```

Добавьте эту функцию (если её ещё нет) в `app.py`.

### 2. Маршруты `GET /pages/new` и `POST /pages/new`

Теперь добавьте маршруты:

```python
@app.get("/pages/new")
def page_new():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_author_or_above():
        abort(403)

    return render_template("page_new.html", error=None)


@app.post("/pages/new")
def page_create():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_author_or_above():
        abort(403)

    title = (request.form.get("title") or "").strip()
    content = request.form.get("content") or ""

    if not title:
        return render_template("page_new.html", error="Введите заголовок страницы.")

    conn = get_conn()

    # Проверка уникальности заголовка
    existing = conn.execute(
        "SELECT id FROM pages WHERE title = ?",
        (title,),
    ).fetchone()
    if existing is not None:
        conn.close()
        return render_template("page_new.html", error="Страница с таким заголовком уже существует.")

    # Генерируем slug и делаем его уникальным
    slug = slugify(title)
    slug = ensure_unique_slug(conn, slug)

    # Создаём страницу
    conn.execute(
        "INSERT INTO pages (title, slug) VALUES (?, ?)",
        (title, slug),
    )
    conn.commit()

    page_row = conn.execute(
        "SELECT id FROM pages WHERE slug = ?",
        (slug,),
    ).fetchone()
    page_id = page_row["id"]

    user = current_user()

    # Создаём первую ревизию, сразу approved
    conn.execute(
        "INSERT INTO revisions (page_id, author_id, content, status) VALUES (?, ?, ?, 'approved')",
        (page_id, user["id"], content),
    )
    conn.commit()
    conn.close()

    flash("Страница создана.")
    return redirect(url_for("page_view", slug=slug))
```

### 3. Шаблон `page_new.html`

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

## E5. Список страниц `/pages`

Сделаем страницу, где выводятся все опубликованные страницы.

### 1. Маршрут `GET /pages`

В `app.py` добавьте:

```python
@app.get("/pages")
def page_list():
    conn = get_conn()
    pages = conn.execute("""
        SELECT
            p.id,
            p.title,
            p.slug,
            p.created_at
        FROM pages p
        WHERE EXISTS (
            SELECT 1
            FROM revisions r
            WHERE r.page_id = p.id AND r.status = 'approved'
        )
        ORDER BY p.title
    """).fetchall()
    conn.close()

    return render_template("page_list.html", pages=pages)
```

### 2. Шаблон `page_list.html`

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

Теперь после создания страниц вы сможете видеть их список и переходить к просмотру по ссылке.

---

## E6. Просмотр одной страницы `/pages/<slug>`

Сделаем маршрут и шаблон, которые показывают актуальное содержимое страницы.

### 1. Маршрут `GET /pages/<slug>`

В `app.py` добавьте:

```python
@app.get("/pages/<slug>")
def page_view(slug):
    page = get_page_by_slug(slug)
    if page is None:
        abort(404)

    conn = get_conn()
    current_rev_id = get_current_revision_id(conn, page["id"])
    revision = None

    if current_rev_id is not None:
        revision = conn.execute(
            """
            SELECT
                r.id,
                r.content,
                r.created_at,
                u.username AS author_username
            FROM revisions r
            LEFT JOIN users u ON r.author_id = u.id
            WHERE r.id = ?
            """,
            (current_rev_id,),
        ).fetchone()

    conn.close()

    return render_template(
        "page_view.html",
        page=page,
        revision=revision,
    )
```

### 2. Шаблон `page_view.html`

Создайте файл `templates/page_view.html`:

```html
{% extends "base.html" %}

{% block title %}{{ page.title }} · Вики{% endblock %}

{% block content %}
  <section class="card">
    <h1>{{ page.title }}</h1>

    {% if revision %}
      <p class="muted">
        {% if revision.author_username %}
          Автор: {{ revision.author_username }}<br />
        {% endif %}
        Последнее обновление: {{ revision.created_at }}
      </p>

      <div class="wiki-content">
        <pre>{{ revision.content }}</pre>
      </div>
    {% else %}
      <p class="muted">
        Для этой страницы пока нет опубликованной ревизии.
      </p>
    {% endif %}

    <p style="margin-top: 16px;">
      <a href="{{ url_for('page_list') }}">К списку страниц</a>
    </p>
  </section>
{% endblock %}
```

На этом шаге мы **не добавляем** ещё форм для редактирования, отправки на проверку и отката — только чтение содержимого.

---

## Итог Части E

После выполнения этой части у вас есть:

- таблица `pages` с заголовком, slug, флагом блокировки и датой создания;
- таблица `revisions` с базовыми полями и статусом `approved`;
- хелперы `slugify`, `ensure_unique_slug`, `get_page_by_slug`, `get_current_revision_id`;
- форма создания страницы `/pages/new`, которая сразу создаёт первую утверждённую ревизию;
- список страниц `/pages` с переходом по slug;
- просмотр одной страницы `/pages/<slug>` с отображением актуального содержимого.

Это первый шаг в предметной логике вики. В следующих частях вы сможете:

- добавить статусы `draft` / `pending` / `rejected` и модерацию правок;
- реализовать историю ревизий и откаты;
- ввести блокировку страниц и расширенное администрирование контента.
{% endraw %}

