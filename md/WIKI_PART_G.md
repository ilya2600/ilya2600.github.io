{% raw %}
# Часть G: Ревизии и создание страницы по роли (черновик vs утверждённая)

В Части F контент страницы хранится в таблице `pages` (поле `content`). В этой части вы:

1. Переносите **содержимое страниц в таблицу ревизий**: страница хранит только метаданные; «текущее» отображаемое содержимое — последняя ревизия со статусом `approved`. Если такой нет — показывается «Нет опубликованной версии».
2. Вводите **создание страницы по роли**: при создании страницы первая ревизия получает статус **`draft`** для автора и **`approved`** для редактора или мастера. В результате страница автора не попадает в публичный список до утверждения; страница редактора/мастера сразу отображается в списке и при просмотре.

**Видимое поведение:** автор создаёт страницу → она **не** появляется в списке страниц; по прямой ссылке видно «Нет опубликованной версии» (черновик). Редактор или мастер создаёт страницу → она **появляется** в списке; при просмотре видны контент и дата ревизии.

---

## G1. Схема БД: таблица `revisions` и удаление `content` из `pages` в `db.py`

### Шаг G1.1. Таблица `revisions`

Откройте `db.py` и в функции `init_db()` после создания таблицы `pages` добавьте создание таблицы ревизий:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS revisions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            page_id INTEGER NOT NULL,
            author_id INTEGER,
            content TEXT NOT NULL DEFAULT '',
            status TEXT NOT NULL CHECK (status IN ('draft', 'pending', 'approved', 'rejected')),
            reviewer_id INTEGER,
            review_note TEXT,
            rollback_reason TEXT,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            FOREIGN KEY (page_id) REFERENCES pages(id),
            FOREIGN KEY (author_id) REFERENCES users(id),
            FOREIGN KEY (reviewer_id) REFERENCES users(id)
        )
    """)
```

Поля: `page_id`, `author_id`, `content`, `status` (draft, pending, approved, rejected), `reviewer_id`, `review_note`, `rollback_reason`, `created_at`.

### Шаг G1.2. Миграция: перенос контента из `pages` в ревизии и удаление столбца `content`

После добавления таблицы `revisions` нужно один раз перенести существующий контент страниц в ревизии и убрать поле `content` из `pages`. Добавьте в `db.py` функцию миграции и вызовите её из `init_db()` после создания таблиц.

**Функция миграции** (вставьте в `db.py` после `get_conn`, перед `init_db`):

```python
def _migrate_content_to_revisions(conn):
    """
    Один раз: для каждой страницы с контентом создать одну ревизию со статусом approved,
    затем удалить столбец content из pages (через пересоздание таблицы).
    """
    cursor = conn.execute("PRAGMA table_info(pages)")
    columns = [row[1] for row in cursor.fetchall()]
    if "content" not in columns:
        return  # уже мигрировано

    # Заполняем revisions из pages
    conn.execute("""
        INSERT INTO revisions (page_id, author_id, content, status, created_at)
        SELECT id, author_id, content, 'approved', created_at
        FROM pages
    """)

    # В SQLite до 3.35 нет DROP COLUMN — пересоздаём таблицу без content
    conn.execute("PRAGMA foreign_keys = OFF")
    conn.execute("""
        CREATE TABLE pages_new (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL UNIQUE,
            slug TEXT NOT NULL UNIQUE,
            author_id INTEGER,
            is_locked INTEGER NOT NULL DEFAULT 0,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            FOREIGN KEY (author_id) REFERENCES users(id)
        )
    """)
    conn.execute("""
        INSERT INTO pages_new (id, title, slug, author_id, is_locked, created_at)
        SELECT id, title, slug, author_id, is_locked, created_at FROM pages
    """)
    conn.execute("DROP TABLE pages")
    conn.execute("ALTER TABLE pages_new RENAME TO pages")
    conn.execute("PRAGMA foreign_keys = ON")
```

В `init_db()` после создания таблицы `pages` добавьте создание `revisions` (блок из шага G1.1), затем **перед `conn.commit()`** вызовите миграцию:

```python
    _migrate_content_to_revisions(conn)
```

**Важно:** если вы только что вводите таблицу `pages` (ещё без данных), в Части F при создании `pages` **не включайте** столбец `content` — оставьте только `id`, `title`, `slug`, `author_id`, `is_locked`, `created_at`. Тогда миграция ничего не сделает, и новые страницы будут создаваться только через ревизии (шаг G2.5).

---

## G2. Логика страниц: контент из последней утверждённой ревизии и создание по роли в `views_pages.py`

### Шаг G2.1. Хелпер: последняя утверждённая ревизия для страницы

Откройте `views_pages.py`. Рядом с `get_page_by_slug` добавьте функцию, возвращающую последнюю по времени ревизию со статусом `approved` для заданного `page_id`:

```python
def get_latest_approved_revision(conn, page_id: int):
    """Вернуть последнюю ревизию страницы со статусом approved или None."""
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
```

### Шаг G2.2. Изменение `get_page_by_slug`: контент из ревизий

Измените `get_page_by_slug` так, чтобы она возвращала данные страницы и **содержимое и дату из последней утверждённой ревизии** (если её нет — поля контента пустые/None):

```python
def get_page_by_slug(slug: str):
    """Найти страницу по slug; контент и дата ревизии — из последней утверждённой ревизии."""
    conn = get_conn()
    page = conn.execute(
        """
        SELECT
            p.id,
            p.title,
            p.slug,
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
```

### Шаг G2.3. Список страниц и главная: только страницы с утверждённой ревизией

Список страниц и просмотр должны быть согласованы: в списке только страницы, у которых есть хотя бы одна ревизия со статусом `approved`. В `page_list_view()` замените запрос на:

```python
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
```

Если на главной странице (home) выводится список страниц, используйте тот же критерий — только страницы с утверждённой ревизией.

### Шаг G2.4. Просмотр страницы

В `page_view_view(slug)` логику менять не нужно: страница запрашивается через `get_page_by_slug` и передаётся в шаблон. Если утверждённой ревизии нет, `page["content"]` и `page["revision_created_at"]` будут `None`; шаблон (G3) выведет «Нет опубликованной версии» и при наличии — дату ревизии «Ревизия: &lt;date&gt;».

### Шаг G2.5. Создание страницы: запись в `pages` без контента и первая ревизия по роли

При создании страницы контент попадает только в таблицу `revisions`. **Статус первой ревизии зависит от роли:**

- **Автор (author)** → статус **`draft`**. Страница не попадает в публичный список; при просмотре по ссылке отображается «Нет опубликованной версии» (черновик).
- **Редактор (editor) или мастер (admin)** → статус **`approved`**. Страница сразу появляется в списке и при просмотре показываются контент и дата ревизии.

В `page_create_view()` в `views_pages.py` замените вставку в `pages` и добавьте вставку ревизии с выбором статуса по роли. После проверки уникальности заголовка и формирования `slug`:

```python
    slug = slugify(title)
    slug = ensure_unique_slug(conn, slug)
    user = current_user()

    cursor = conn.execute(
        "INSERT INTO pages (title, slug, author_id) VALUES (?, ?, ?)",
        (title, slug, user["id"]),
    )
    page_id = cursor.lastrowid

    # Первая ревизия: автор — черновик, редактор или мастер — сразу утверждённая
    if user["role"] in ("editor", "admin"):
        status = "approved"
    else:
        status = "draft"

    conn.execute(
        """INSERT INTO revisions (page_id, author_id, content, status) VALUES (?, ?, ?, ?)""",
        (page_id, user["id"], content, status),
    )
    conn.commit()
    conn.close()

    flash("Страница создана.")
    return redirect(url_for("page_view", slug=slug))
```

В результате при создании страницы автором она не появится в списке и при открытии по URL будет показано «Нет опубликованной версии»; при создании редактором или мастером страница сразу отображается в списке и с контентом при просмотре.

---

## G3. Шаблон просмотра страницы: дата ревизии и «нет опубликованной версии»

Откройте `templates/page_view.html`. Блок с контентом и метаданными замените на условный вывод: при наличии утверждённой ревизии показывать контент и дату ревизии («Ревизия: &lt;date&gt;»); при отсутствии — «Нет опубликованной версии».

Пример:

```html
    <p class="muted">
      {% if page.author_username %}
        Автор: {{ page.author_username }}<br />
      {% endif %}
      Создана: {{ page.created_at }}
      {% if page.revision_created_at %}
        · Ревизия: {{ page.revision_created_at }}
      {% endif %}
    </p>

    <div class="wiki-content">
      {% if page.content is not none and page.content != '' %}
        <pre>{{ page.content }}</pre>
      {% else %}
        <p class="muted">Нет опубликованной версии.</p>
      {% endif %}
    </div>
```

Так при наличии утверждённой ревизии отображаются «Ревизия: &lt;date&gt;» и контент; при отсутствии — «Нет опубликованной версии».

---

## Visible / testable (что можно проверить)

- **Автор:** создать страницу → она **не** появляется в списке страниц; открытие по URL (/pages/&lt;slug&gt;) показывает «Нет опубликованной версии».
- **Редактор или мастер:** создать страницу → она **появляется** в списке; при просмотре видны контент и «Ревизия: &lt;date&gt;».
- На любой странице с утверждённой ревизией отображаются дата ревизии и контент; без утверждённой ревизии — «Нет опубликованной версии».

---

## Итог Части G

После выполнения части G:

- **`db.py`**: таблица `revisions` (page_id, author_id, content, status, reviewer_id, review_note, rollback_reason, created_at) со статусами draft, pending, approved, rejected. Таблица `pages` без столбца `content`. Миграция `_migrate_content_to_revisions` переносит существующий контент в одну утверждённую ревизию на страницу и убирает столбец `content`.
- **`views_pages.py`**: хелпер `get_latest_approved_revision(conn, page_id)`; `get_page_by_slug` возвращает страницу с контентом и датой из последней утверждённой ревизии (или None); список страниц — только страницы с хотя бы одной ревизией со статусом `approved`; создание страницы — вставка в `pages` и одна ревизия: **author → `draft`**, **editor или admin → `approved`**.
- **`templates/page_view.html`**: вывод контента и при наличии — «Ревизия: &lt;date&gt;»; при отсутствии утверждённой ревизии — «Нет опубликованной версии».

Контент хранится только в ревизиях; «текущее» содержимое страницы = последняя ревизия со статусом `approved`. Создание страницы по роли даёт видимое различие: страница автора не в списке и без опубликованной версии; страница редактора/мастера сразу в списке и с контентом. В следующих частях — правка страницы (новая ревизия), отправка черновика на проверку, утверждение и отклонение.
{% endraw %}
