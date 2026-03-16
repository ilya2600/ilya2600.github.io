{% raw %}
# Часть J: История ревизий, просмотр ревизии и откат

После Части I доступны просмотр ревизии (GET `/revisions/<id>`), утверждение/отклонение и дашборд. В этой части вы добавляете:

1. **История ревизий страницы:** маршрут `/pages/<slug>/revisions` — список всех ревизий страницы (новые сверху): id, статус, дата, автор, рецензент, заметка рецензента, причина отката. У каждой строки — ссылка **«Просмотр»** на страницу просмотра ревизии (тот же маршрут, что в Части I). **Редакторы и мастера** видят ревизии всех статусов; **остальные** — только утверждённые. На странице просмотра страницы — ссылка «История».
2. **Просмотр ревизии:** используется та же страница «Просмотр ревизии», что и в Части I. Из истории по ссылке «Просмотр» открывается содержимое выбранной ревизии; правила видимости не меняются (редактор видит все статусы в списке и может открыть любую; остальные видят только утверждённые и могут открыть их).
3. **Откат:** только редактор или мастер. Форма выбора **прошлой утверждённой** ревизии, необязательное поле «Причина отката»; отправка → создаётся **новая** ревизия с контентом выбранной, статус `approved`. На странице просмотра страницы — ссылка «Откат» (только для редакторов и мастеров).

**Видимо:** по ссылке «История» открывается список ревизий; у каждой — «Просмотр» → отображается **содержимое** этой ревизии. Редактор видит черновики/ожидающие/отклонённые и может открыть их содержимое; остальные — только утверждённые. По ссылке «Откат» редактор выбирает версию и отправляет форму → контент страницы становится как у выбранной версии; в истории появляется новая ревизия.

---

## J1. История ревизий страницы (маршрут и представление)

### Шаг J1.1. Правила видимости списка ревизий

- **Редактор или мастер:** видит в списке ревизии **всех** статусов (draft, pending, approved, rejected).
- **Автор и остальные:** видят только ревизии со статусом **approved**.

Ссылка «Просмотр» ведёт на тот же маршрут GET `/revisions/<revision_id>` и шаблон `revision_view.html` (Часть I). Права на просмотр каждой ревизии уже заданы в Части I: редактор может открыть любую; автор — свою (draft/pending) или любую утверждённую. Поэтому в списке истории редактор видит все строки и может открыть любую по «Просмотр»; остальные видят только утверждённые и могут открыть только их.

### Шаг J1.2. Представление и маршрут истории ревизий

В `views_pages.py` добавьте представление, которое по `slug` страницы загружает страницу и список ревизий с полями: id, status, created_at, author_username, reviewer_username, review_note, rollback_reason. Редактору отдайте все ревизии; остальным — только со статусом `approved`. Сортировка: сначала новые (ORDER BY created_at DESC).

Пример запроса для списка ревизий (редактор — все, не-редактор — только approved):

```python
def page_revisions_view(slug: str):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_author_or_above():
        abort(403)

    conn = get_conn()
    page = conn.execute(
        "SELECT id, title, slug FROM pages WHERE slug = ?", (slug,)
    ).fetchone()
    if page is None:
        conn.close()
        abort(404)
    page = dict(page)

    user = current_user()
    if is_editor():
        rows = conn.execute(
            """
            SELECT r.id, r.status, r.created_at, r.review_note, r.rollback_reason,
                   u1.username AS author_username, u2.username AS reviewer_username
            FROM revisions r
            LEFT JOIN users u1 ON r.author_id = u1.id
            LEFT JOIN users u2 ON r.reviewer_id = u2.id
            WHERE r.page_id = ?
            ORDER BY r.created_at DESC
            """,
            (page["id"],),
        ).fetchall()
    else:
        rows = conn.execute(
            """
            SELECT r.id, r.status, r.created_at, r.review_note, r.rollback_reason,
                   u1.username AS author_username, u2.username AS reviewer_username
            FROM revisions r
            LEFT JOIN users u1 ON r.author_id = u1.id
            LEFT JOIN users u2 ON r.reviewer_id = u2.id
            WHERE r.page_id = ? AND r.status = 'approved'
            ORDER BY r.created_at DESC
            """,
            (page["id"],),
        ).fetchall()

    revisions = [dict(row) for row in rows]
    conn.close()

    return render_template(
        "page_revisions.html",
        page=page,
        revisions=revisions,
    )
```

В `app.py` добавьте маршрут GET **до** маршрута `/pages/<slug>` (чтобы путь `.../revisions` не воспринимался как slug):

```python
@app.get("/pages/<slug>/revisions")
def page_revisions(slug):
    return page_revisions_view(slug)
```

Импортируйте `page_revisions_view` из `views_pages`.

### Шаг J1.3. Шаблон истории ревизий

Создайте `templates/page_revisions.html`: заголовок страницы (например, «История ревизий: {{ page.title }}»), таблица ревизий — колонки: id (или «№»), статус, дата, автор, рецензент, заметка рецензента, причина отката, действие. В колонке «Действие» — ссылка **«Просмотр»** на `url_for('revision_view', revision_id=rev.id)`. Внизу — ссылка «К странице» на `page_view` по slug.

Пример фрагмента таблицы:

```html
<table class="table">
  <thead>
    <tr>
      <th>№</th>
      <th>Статус</th>
      <th>Дата</th>
      <th>Автор</th>
      <th>Рецензент</th>
      <th>Заметка</th>
      <th>Причина отката</th>
      <th>Действие</th>
    </tr>
  </thead>
  <tbody>
    {% for rev in revisions %}
      <tr>
        <td>{{ rev.id }}</td>
        <td>{{ rev.status }}</td>
        <td>{{ rev.created_at[:10] if rev.created_at else '' }}</td>
        <td>{{ rev.author_username or '—' }}</td>
        <td>{{ rev.reviewer_username or '—' }}</td>
        <td>{{ rev.review_note or '—' }}</td>
        <td>{{ rev.rollback_reason or '—' }}</td>
        <td>
          <a href="{{ url_for('revision_view', revision_id=rev.id) }}" class="btn btn-sm">Просмотр</a>
        </td>
      </tr>
    {% endfor %}
  </tbody>
</table>
```

### Шаг J1.4. Ссылка «История» на странице просмотра страницы

В `templates/page_view.html` добавьте ссылку «История» (например, рядом с «К списку страниц»), ведущую на `url_for('page_revisions', slug=page.slug)`.

Пример:

```html
    <p style="margin-top: 16px;">
      <a href="{{ url_for('page_revisions', slug=page.slug) }}">История</a>
      ·
      <a href="{{ url_for('page_list') }}">К списку страниц</a>
    </p>
```

---

## J2. Просмотр ревизии из истории

Дополнительная реализация не требуется. Ссылка «Просмотр» в истории ведёт на тот же маршрут GET `/revisions/<revision_id>` и представление `revision_view_view` из Части I. Пользователь открывает выбранную ревизию и видит её содержимое; правила видимости (редактор — любая ревизия; автор — своя или утверждённая) уже реализованы в Части I.

---

## J3. Откат к выбранной утверждённой ревизии

Откат доступен только редактору и мастеру. В результате отката создаётся **новая** ревизия: контент копируется из выбранной **утверждённой** ревизии, статус новой ревизии — `approved`; при необходимости заполняется поле `rollback_reason`. Старые ревизии не изменяются.

### Шаг J3.1. Представление формы отката (GET)

В `views_pages.py` добавьте представление для страницы формы отката: по `slug` загружается страница и список **только утверждённых** ревизий этой страницы (для выбора в форме), например с полями id, created_at, author_username. Передайте в шаблон страницу и список `approved_revisions`.

Пример:

```python
def page_rollback_form_view(slug: str):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_editor():
        abort(403)

    conn = get_conn()
    page = conn.execute(
        "SELECT id, title, slug FROM pages WHERE slug = ?", (slug,)
    ).fetchone()
    if page is None:
        conn.close()
        abort(404)
    page = dict(page)

    rows = conn.execute(
        """
        SELECT r.id, r.created_at, u.username AS author_username
        FROM revisions r
        LEFT JOIN users u ON r.author_id = u.id
        WHERE r.page_id = ? AND r.status = 'approved'
        ORDER BY r.created_at DESC
        """,
        (page["id"],),
    ).fetchall()
    approved_revisions = [dict(row) for row in rows]
    conn.close()

    return render_template(
        "page_rollback.html",
        page=page,
        approved_revisions=approved_revisions,
    )
```

### Шаг J3.2. Представление обработки отката (POST)

Добавьте представление, которое принимает `slug`, `revision_id` (id выбранной утверждённой ревизии) и необязательное поле `rollback_reason` из формы. Проверка: пользователь — редактор или мастер; страница существует; ревизия существует, принадлежит этой странице и имеет статус `approved`. Затем создаётся **новая** ревизия: `page_id`, `author_id` = текущий пользователь, `content` = контент выбранной ревизии, `status` = `approved`, `rollback_reason` = значение из формы (или None). После вставки — редирект на страницу просмотра и flash «Откат выполнен».

Пример:

```python
def page_rollback_submit_view(slug: str):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_editor():
        abort(403)

    conn = get_conn()
    page = conn.execute(
        "SELECT id, title, slug FROM pages WHERE slug = ?", (slug,)
    ).fetchone()
    if page is None:
        conn.close()
        abort(404)

    revision_id = request.form.get("revision_id")
    if not revision_id:
        conn.close()
        abort(400)
    try:
        revision_id = int(revision_id)
    except ValueError:
        conn.close()
        abort(400)

    rev = conn.execute(
        "SELECT id, page_id, content, status FROM revisions WHERE id = ? AND page_id = ?",
        (revision_id, page["id"]),
    ).fetchone()
    if rev is None or rev["status"] != "approved":
        conn.close()
        abort(404)

    user = current_user()
    rollback_reason = (request.form.get("rollback_reason") or "").strip() or None

    conn.execute(
        """
        INSERT INTO revisions (page_id, author_id, content, status, rollback_reason)
        VALUES (?, ?, ?, 'approved', ?)
        """,
        (page["id"], user["id"], rev["content"], rollback_reason),
    )
    conn.commit()
    conn.close()

    flash("Откат выполнен.")
    return redirect(url_for("page_view", slug=slug))
```

### Шаг J3.3. Маршруты отката в `app.py`

Добавьте маршруты **до** маршрута `/pages/<slug>`:

```python
@app.get("/pages/<slug>/rollback")
def page_rollback_form(slug):
    return page_rollback_form_view(slug)


@app.post("/pages/<slug>/rollback")
def page_rollback_submit(slug):
    return page_rollback_submit_view(slug)
```

Импортируйте `page_rollback_form_view` и `page_rollback_submit_view` из `views_pages`.

### Шаг J3.4. Шаблон формы отката

Создайте `templates/page_rollback.html`: заголовок («Откат: {{ page.title }}»), форма с полем выбора ревизии (например, `<select name="revision_id">` — варианты из `approved_revisions`, value = id, подпись — дата и/или автор), необязательное поле «Причина отката» (`name="rollback_reason"`), кнопка «Выполнить откат». Метод формы — POST, action — `url_for('page_rollback_submit', slug=page.slug)`. Ссылка «Отмена» или «К странице» на `page_view`.

Пример:

```html
{% extends "base.html" %}

{% block title %}Откат: {{ page.title }} · Вики{% endblock %}

{% block content %}
  <section class="card">
    <h1>Откат</h1>
    <p><a href="{{ url_for('page_view', slug=page.slug) }}">{{ page.title }}</a></p>

    <form method="post" action="{{ url_for('page_rollback_submit', slug=page.slug) }}">
      <p>
        <label for="revision_id">Версия для отката:</label>
        <select name="revision_id" id="revision_id" required>
          {% for rev in approved_revisions %}
            <option value="{{ rev.id }}">Ревизия #{{ rev.id }} · {{ rev.created_at or '—' }} · автор: {{ rev.author_username or '—' }}</option>
          {% endfor %}
        </select>
      </p>
      <p>
        <label for="rollback_reason">Причина отката (необязательно):</label>
        <input type="text" name="rollback_reason" id="rollback_reason" size="40" placeholder="Необязательно" />
      </p>
      <p>
        <button type="submit" class="btn btn-primary">Выполнить откат</button>
        <a href="{{ url_for('page_view', slug=page.slug) }}" class="btn">Отмена</a>
      </p>
    </form>
  </section>
{% endblock %}
```

### Шаг J3.5. Ссылка «Откат» на странице просмотра страницы

В `templates/page_view.html` добавьте ссылку «Откат» для редакторов и мастеров (например, рядом с «История»), ведущую на `url_for('page_rollback_form', slug=page.slug)`. Отображать только при `is_editor()`.

Пример:

```html
    <p style="margin-top: 16px;">
      <a href="{{ url_for('page_revisions', slug=page.slug) }}">История</a>
      {% if is_editor() %}
        · <a href="{{ url_for('page_rollback_form', slug=page.slug) }}">Откат</a>
      {% endif %}
      · <a href="{{ url_for('page_list') }}">К списку страниц</a>
    </p>
```

---

## Visible / testable (что можно проверить)

- **Любой авторизованный автор и выше:** На странице просмотра есть ссылка «История»; по ней открывается список ревизий; в списке только утверждённые ревизии; у каждой — «Просмотр» → открывается страница с **содержимым** этой ревизии.
- **Редактор или мастер:** В истории видны ревизии всех статусов (draft, pending, approved, rejected); по «Просмотр» можно открыть любую и увидеть её содержимое. На странице просмотра страницы есть ссылка «Откат»; по ней — форма выбора утверждённой версии и необязательная причина отката; после отправки контент страницы совпадает с выбранной версией; в истории появляется новая ревизия.
- **Автор (не редактор):** Ссылки «Откат» на странице просмотра нет.

---

## Итог Части J

После выполнения части J:

- **История ревизий:** маршрут GET `/pages/<slug>/revisions`, представление `page_revisions_view(slug)` с фильтром по статусу (редактор — все, остальные — только approved), шаблон `page_revisions.html` с таблицей ревизий (id, статус, дата, автор, рецензент, заметка, причина отката) и ссылкой «Просмотр» на страницу просмотра ревизии. На странице просмотра страницы — ссылка «История».
- **Просмотр ревизии:** без изменений — используется маршрут и представление из Части I; из истории по «Просмотр» открывается содержимое выбранной ревизии.
- **Откат:** маршруты GET и POST `/pages/<slug>/rollback`, представления `page_rollback_form_view(slug)` и `page_rollback_submit_view(slug)` (только для редактора/мастера); создаётся новая ревизия с контентом выбранной утверждённой и статусом approved; шаблон `page_rollback.html`. На странице просмотра страницы — ссылка «Откат» (только при `is_editor()`).

В результате история ревизий прозрачна, любая ревизия из списка может быть открыта для просмотра содержимого; откат выполняется созданием новой утверждённой ревизии и не меняет старые записи.
{% endraw %}
