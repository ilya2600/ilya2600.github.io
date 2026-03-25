# Отчет: критические проблемы (падения/ошибки) и готовые исправления

Проект: небольшой Flask + SQLite (Kanban) веб-сервис.

Ниже перечислены **“severe”** проблемы, которые с высокой вероятностью вызывают падения приложения (ошибки импорта/синтаксиса/исключения в рантайме) либо гарантированно ломают функциональность при обращении к соответствующим маршрутам.

---

## 0) Как применять правки (шаблон “сделай раз-два-три”)

- **Шаг 1**: Откройте файл, указанный в разделе проблемы (например `views_auth.py`).
- **Шаг 2**: Найдите в файле кусок кода, который в документе помечен как “НАЙДИТЕ ЭТО”.
- **Шаг 3**: Замените его на “ВСТАВЬТЕ ЭТО”.
- **Шаг 4**: Сохраните файл.
- **Шаг 5**: Запустите проверку:

```bash
python app.py
```

---

## 1) Краткое резюме (что сломано в первую очередь)

1. `views_pages.py` — **несовместим с импортом** (отсутствуют импорты, есть синтаксическая ошибка/неправильная индентификация, функция неполная).
2. `views_auth.py` / `auth_utils.py` — логика проверки админа содержит **ошибку доступа к полям `sqlite3.Row`** (`.get()` у `Row` нет), что приводит к падению при проверке `is_admin`.
3. `views_boards.py` — `card_edit_view` выполняет **неправильный SQL**: в `UPDATE cards SET ...` подставляются **title/description вместо `column_id/position`**, что вызывает ошибки БД и/или порчу данных.
4. `views_boards.py` — `board_view_view` содержит **заглушку** (“Test Board”), перезатирающую реальные данные, что ломает отображение.
5. `db.py` — конфигурация БД и схема содержат **грубые несоответствия**: отсутствует создание таблицы `pages`, но она используется в ограничениях FK/логике; также подключение к БД использует **плейсхолдер** `your_database.db` и содержит **несвойственный Flask код**.


---

## 2) Подробные “severe” проблемы по файлам

### `views_pages.py` — синтаксис + отсутствие импортов + неполная реализация

**Симптомы**
- При попытке импортировать модуль `views_pages.py` приложение падает с `SyntaxError`/`IndentationError`.
- Даже без импорта: функции используют `get_conn`, `render_template`, `slugify`, `ensure_unique_slug`, `current_user`, `flash`, `url_for`, переменные `title/content/conn/...` — но **никаких импортов и переменных в модуле нет**.

**Что именно в коде**
- Файл начинается с `def get_page_by_slug(...):` и **не содержит импортов**.
- Функция `page_create_view` объявлена как `def page_create_view()` без двоеточия `:` и дальше тело неверно индентирано.

**Как исправить (варианты)**

#### Вариант A (самый простой): если “страницы” не используются — отключить файл

По сканированию проекта `views_pages.py` **нигде не импортируется**. Поэтому проще всего:
- либо удалить `views_pages.py`,
- либо переименовать его в `views_pages_UNUSED.py`.

Так вы убираете “мину” (синтаксические ошибки) и не тратите время на недописанный функционал.

#### Вариант B (если “страницы” нужны): вставить минимально рабочую версию `views_pages.py`

Сделайте так:
- откройте `views_pages.py`
- **удалите содержимое файла целиком**
- **вставьте** нижеуказанный код полностью

ВСТАВЬТЕ ЭТО (полностью):

```python
from flask import render_template, request, redirect, url_for, flash
from db import get_conn
from auth_utils import is_logged_in, current_user
import re


def _slugify(text: str) -> str:
    text = (text or "").strip().lower()
    text = re.sub(r"\s+", "-", text)
    text = re.sub(r"[^a-z0-9\-а-яё]", "", text)
    text = re.sub(r"-{2,}", "-", text).strip("-")
    return text or "page"


def _ensure_unique_slug(conn, slug: str) -> str:
    base = slug
    i = 1
    while True:
        row = conn.execute("SELECT 1 FROM pages WHERE slug = ? LIMIT 1", (slug,)).fetchone()
        if row is None:
            return slug
        i += 1
        slug = f"{base}-{i}"


def get_latest_approved_revision(conn, page_id: int):
    return conn.execute(
        """
        SELECT id, page_id, author_id, content, status, reviewer_id, review_note, rollback_reason, created_at
        FROM revisions
        WHERE page_id = ? AND status = 'approved'
        ORDER BY created_at DESC
        LIMIT 1
        """,
        (page_id,),
    ).fetchone()


def get_page_by_slug(slug: str):
    conn = get_conn()
    page = conn.execute(
        """
        SELECT
            p.id, p.title, p.slug, p.author_id, p.is_locked, p.created_at,
            u.username AS author_username
        FROM pages p
        LEFT JOIN users u ON p.author_id = u.id
        WHERE p.slug = ?
        """,
        (slug,),
    ).fetchone()
    if page is None:
        conn.close()
        return None

    page = dict(page)
    rev = get_latest_approved_revision(conn, page["id"])
    conn.close()

    if rev is not None:
        page["content"] = rev["content"]
        page["revision_created_at"] = rev["created_at"]
    else:
        page["content"] = None
        page["revision_created_at"] = None
    return page


def page_list_view():
    conn = get_conn()
    pages = conn.execute(
        """
        SELECT DISTINCT p.id, p.title, p.slug, p.created_at
        FROM pages p
        INNER JOIN revisions r ON r.page_id = p.id AND r.status = 'approved'
        ORDER BY p.title
        """
    ).fetchall()
    conn.close()
    return render_template("page_list.html", pages=pages)


def page_create_view():
    if not is_logged_in():
        return redirect(url_for("login_form"))

    title = (request.form.get("title") or "").strip()
    content = request.form.get("content") or ""
    if not title:
        return render_template("page_new.html", error="Введите заголовок.")

    conn = get_conn()
    slug = _ensure_unique_slug(conn, _slugify(title))
    user = current_user()

    cur = conn.execute(
        "INSERT INTO pages (title, slug, author_id) VALUES (?, ?, ?)",
        (title, slug, user["id"]),
    )
    page_id = cur.lastrowid

    status = "approved" if user["role"] == "admin" else "draft"
    conn.execute(
        "INSERT INTO revisions (page_id, author_id, content, status) VALUES (?, ?, ?, ?)",
        (page_id, user["id"], content, status),
    )
    conn.commit()
    conn.close()

    flash("Страница создана.")
    return redirect(url_for("page_view", slug=slug))
```

Если вы выбрали Вариант B, обязательно выполните исправление из раздела `db.py → (A) pages`, иначе будет ошибка `no such table: pages`.

---

### `auth_utils.py` (или `views_auth.py`, где используется `is_admin`) — `sqlite3.Row` не имеет `.get()`

**Симптомы**
- Падение с `AttributeError: 'sqlite3.Row' object has no attribute 'get'` при вызове `is_admin()`.
- Часто всплывает при заходе на страницу администратора или при обращении к `is_admin` из шаблонов (`home.html`, `dashboard.html` и т.п.).

**Что именно**
Функция `is_admin()` обращается к роли так:
- `user.get("role")`

Но `current_user()` возвращает `sqlite3.Row`, у которого нет `.get()`.

**Как исправить (копипаст)**

Откройте файл `views_auth.py` и найдите функцию `is_admin`.

НАЙДИТЕ ЭТО:

```python
def is_admin():
    user = current_user()
    return user is not None and user.get("role") == "admin"
```

ВСТАВЬТЕ ЭТО:

```python
def is_admin():
    user = current_user()
    return user is not None and user["role"] == "admin"
```

Почему это работает: `current_user()` возвращает `sqlite3.Row`, у него доступ к колонкам через `row["role"]`, а не через `.get(...)`.

---

### `views_boards.py` — `card_edit_view`: неправильный SQL и неправильные параметры

**Симптомы**
- При сохранении правок карточки приложение падает с ошибкой SQL (или “тихо” портит данные).
- Типичный эффект: в `cards.column_id` и `cards.position` записываются строки из `title/description`.

**Что именно**
В `card_edit_view` выполняется SQL:

```sql
UPDATE cards
SET column_id = ?, position = ?, updated_at = ...
WHERE id = ?
```

Но в параметры передается:
`(title, description or "", card_id)`

То есть в `column_id` уходит `title`, а в `position` уходит `description`.

**Как исправить (копипаст)**

Откройте `views_boards.py`, найдите функцию `card_edit_view` и внутри неё найдите блок `UPDATE cards`.

НАЙДИТЕ ЭТО:

```python
    conn.execute(
        """
        UPDATE cards
        SET column_id = ?, position = ?, updated_at = (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        WHERE id = ?
        """,
        (title, description or "", card_id),
    )
```

ВСТАВЬТЕ ЭТО:

```python
    conn.execute(
        """
        UPDATE cards
        SET title = ?, description = ?, updated_at = (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        WHERE id = ?
        """,
        (title, description or "", card_id),
    )
```

Дальше в конце функции исправьте редирект, потому что там используется переменная `board_id`, которая **не определена**.

НАЙДИТЕ ЭТО:

```python
    return redirect(url_for("board_view", board_id=board_id))
```

ВСТАВЬТЕ ЭТО:

```python
    return redirect(url_for("board_view", board_id=card["board_id"]))
```

---

### `views_boards.py` — `board_view_view`: заглушка “Test Board”, затирающая реальные данные

**Симптомы**
- Страница доски показывает неверные данные (или ломает шаблон, если ожидаются конкретные структуры `columns/collaborators/activity`).
- Фактические данные доски игнорируются из-за “Test Board”.

**Что именно**
В `board_view_view` есть блок:
- `columns = []`
- `can_edit = True`
- `collaborators = []`
- `board = {"id": board_id, "title": "Test Board"}`

**Как исправить**
**Как исправить (минимально рабочая версия, копипаст)**

Откройте `views_boards.py`, найдите функцию `board_view_view` и **замените её целиком** на код ниже.

ВСТАВЬТЕ ЭТО (целиком функция):

```python
def board_view_view(board_id: int):
    conn = get_conn()
    board = conn.execute("SELECT * FROM boards WHERE id = ?", (board_id,)).fetchone()
    if board is None:
        conn.close()
        abort(404)

    if not can_view_board(board_id):
        conn.close()
        abort(404)

    columns = conn.execute(
        """
        SELECT id, board_id, title, position, wip_limit
        FROM columns
        WHERE board_id = ?
        ORDER BY position ASC, id ASC
        """,
        (board_id,),
    ).fetchall()
    conn.close()

    cards_by_column = cards_for_board(board_id)
    collaborators = []  # можно добавить позже
    activity = recent_activity_for_board(board_id)
    can_edit = can_edit_board(board_id)

    return render_template(
        "board.html",
        board=board,
        columns=columns,
        cards_by_column=cards_by_column,
        collaborators=collaborators,
        activity=activity,
        is_owner=is_board_owner(board_id),
        can_edit=can_edit,
    )
```

Если шаблон `board.html` ожидает словарь, замените `board=board` на `board=dict(board)`.

---

### `db.py` — схема и подключение к БД содержат грубые несоответствия

#### (A) `pages` таблица не создается, но используется

**Симптомы**
- Любые запросы/логика, работающие с `pages`, упадут с `no such table: pages`.
- Даже если “сейчас нет маршрутов”, вы получили “мина” в схеме: FK и миграции завязаны на `pages`.

**Что именно**
- В `init_db()` создаются `users`, `boards`, `columns`, `revisions`, `cards`, `collaborators`, `card_activity`, `settings`.
- Таблица `pages` в `init_db()` отсутствует.
- При этом есть:
  - FK: `revisions.page_id` ссылается на `pages(id)`
  - миграция `_migrate_content_to_revisions()` смотрит `PRAGMA table_info(pages)`

**Как исправить**
**Как исправить (копипаст)**

Откройте `db.py`, найдите функцию `init_db()` и вставьте создание таблицы `pages`.

ВСТАВЬТЕ ЭТО в `init_db()` (например, сразу после создания таблицы `columns`):

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS pages (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL UNIQUE,
            slug TEXT NOT NULL UNIQUE,
            author_id INTEGER,
            is_locked INTEGER NOT NULL DEFAULT 0,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            FOREIGN KEY (author_id) REFERENCES users(id)
        )
    """)
```

#### (B) `get_conn()` подключается к `your_database.db` (плейсхолдер), игнорируя `DB_PATH`

**Симптомы**
- Приложение может создавать таблицы/данные в “не том” файле.
- Пользователь думает, что БД лежит как `database.db`, но приложение пишет в `your_database.db`.

**Что именно**
- `DB_PATH = "database.db"`
- но `sqlite3.connect('your_database.db', ...)`

**Как исправить**
**Как исправить (копипаст)**

Откройте `db.py`, найдите `DB_PATH = "database.db"` и функцию `get_conn()`.

НАЙДИТЕ ЭТО:

```python
DB_PATH = "database.db"

def get_conn():
    conn = sqlite3.connect('your_database.db', timeout=15.0, check_same_thread=False)
```

ВСТАВЬТЕ ЭТО:

```python
import os

DB_PATH = os.environ.get("KANBAN_DB_PATH", "database.db")

def get_conn():
    conn = sqlite3.connect(DB_PATH, timeout=15.0, check_same_thread=False)
```

PowerShell пример (на время текущей сессии терминала):

```powershell
$env:KANBAN_DB_PATH="database.db"
python app.py
```

#### (C) В `db.py` есть Flask `app` (не нужно в модуле БД)

**Симптомы**
- Это архитектурно неверно (DB-модуль не должен поднимать Flask-приложение).
- Может привести к путанице/несовместимости при дальнейшем развитии.

**Как исправить**
**Как исправить (копипаст)**

Откройте `db.py`. В самом начале сейчас есть Flask-импорты и создание `app`.

НАЙДИТЕ ЭТО (или очень похожее):

```python
from flask import Flask, render_template,session,request,url_for, redirect,abort,flash
import sqlite3

app = Flask(__name__)
```

ЗАМЕНИТЕ НА ЭТО:

```python
import sqlite3
import os
```

---

---

## 3) Рекомендуемый порядок исправлений (минимум усилий, максимум стабильности)

1. Исправить `is_admin()` (замена `.get("role")` на `["role"]`).
2. Починить `card_edit_view` SQL (обновлять `title/description`, а не `column_id/position`).
3. Удалить заглушку `board_view_view` и вернуть реальную выборку данных для шаблона.
4. Привести `db.py`:
   - исправить путь БД на `DB_PATH`,
   - добавить `pages` таблицу в `init_db()`,
   - убрать Flask-конфиги из `db.py`.
5. (Опционально, но полезно) Починить `views_pages.py` полностью или удалить, если оно не используется.


---

## 4) Быстрый чек после правок

1. Запустить приложение:

```bash
python app.py
```

2. Если появляется `no such table: ...`:
- Вы, скорее всего, запускали проект с другой БД (`your_database.db` vs `database.db`) или с “поломанной” схемой.
- Для учебного проекта проще всего удалить файл БД и дать приложению создать схему заново.

⚠️ Важно: удаление БД удалит данные.

3. Пройти вручную в браузере:
- вход/регистрация
- просмотр dashboard
- доступ к админ-страницам (не должно падать)
- редактирование карточки (должно менять `title/description`, не должно менять `column_id/position`)
- просмотр доски (должны отображаться реальные данные, а не “Test Board”)

4. Если включали “pages” (вариант B для `views_pages.py`):
- убедитесь, что в `db.py` добавлена таблица `pages` (раздел `db.py → (A)`).

