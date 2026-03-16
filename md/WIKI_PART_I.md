{% raw %}
# Часть I (полная): Утверждение и отклонение ревизий, очередь ожидающих и просмотр ревизии

После Части H ревизии могут быть в статусе `pending`. В этой части вы добавляете:

1. **Просмотр ревизии:** отдельный маршрут GET, который показывает одну ревизию — заголовок страницы, статус, автор, дата и **текст ревизии** (только чтение). Редактор видит содержимое ожидающей ревизии и на той же странице может нажать «Утвердить» или «Отклонить». Автор видит содержимое своей отправленной на проверку ревизии.
2. **Утверждение и отклонение ревизии:** только редактор или мастер переводит ревизию из `pending` в `approved` (с записью `reviewer_id`) или в `rejected` (с `reviewer_id` и необязательной `review_note`).
3. **Дашборд:** для редакторов — блок «Очередь ожидающих» со списком ревизий в статусе `pending`, ссылкой **«Просмотр»** и кнопками «Утвердить» / «Отклонить». Для авторов — блок «Мои правки на проверке» со ссылкой **«Просмотр»** на содержимое каждой ревизии.

**Видимо:** редактор в очереди нажимает «Просмотр» → видит текст ревизии → утверждает или отклоняет с той же страницы. Автор в «Мои правки на проверке» нажимает «Просмотр» → видит то, что отправил на проверку.

---

## I1. Просмотр одной ревизии (маршрут и представление)

Чтобы и редактор, и автор видели **содержимое** ревизии (а не только метаданные), вводится страница «Просмотр ревизии».

### Шаг I1.1. Правила видимости

- **Редактор или мастер:** может открыть любую ревизию (pending, approved, rejected, draft) — чтобы модерировать и просматривать историю.
- **Автор:** может открыть свою ревизию (draft или pending) и любую утверждённую (approved) ревизию. Чужие черновики и ожидающие/отклонённые автор не видит.

### Шаг I1.2. Представление и маршрут просмотра ревизии

В `views_pages.py` добавьте представление, которое по `revision_id` загружает ревизию вместе со страницей и автором (username), проверяет права по правилам выше и отдаёт шаблон с полями ревизии (включая `content`). Если ревизии нет или доступ запрещён — 404 или 403.

Передайте в шаблон: данные ревизии (id, content, status, created_at), страницу (title, slug), имя автора ревизии, флаг `can_approve_reject` (True только если текущий пользователь — редактор/мастер и статус ревизии — `pending`), чтобы на странице просмотра показывать формы «Утвердить» / «Отклонить» только в этом случае.

Пример реализации представления:

```python
def revision_view_view(revision_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_author_or_above():
        abort(403)

    conn = get_conn()
    rev = conn.execute(
        """
        SELECT r.id, r.page_id, r.author_id, r.content, r.status, r.reviewer_id, r.review_note, r.created_at,
               p.title AS page_title, p.slug AS page_slug
        FROM revisions r
        JOIN pages p ON r.page_id = p.id
        WHERE r.id = ?
        """,
        (revision_id,),
    ).fetchone()
    if rev is None:
        conn.close()
        abort(404)

    rev = dict(rev)
    author = conn.execute("SELECT username FROM users WHERE id = ?", (rev["author_id"],)).fetchone() if rev.get("author_id") else None
    rev["author_username"] = author["username"] if author else "—"
    conn.close()

    user = current_user()
    if is_editor():
        can_view = True
    else:
        can_view = (rev["author_id"] == user["id"]) or (rev["status"] == "approved")
    if not can_view:
        abort(403)

    can_approve_reject = is_editor() and rev["status"] == "pending"

    return render_template(
        "revision_view.html",
        revision=rev,
        can_approve_reject=can_approve_reject,
    )
```

В `app.py` добавьте GET-маршрут просмотра ревизии **рядом с остальными маршрутами ревизий** (например, рядом с `revision_submit`, `revision_approve`, `revision_reject`), чтобы порядок маршрутов был предсказуемым:

```python
@app.get("/revisions/<int:revision_id>")
def revision_view(revision_id):
    return revision_view_view(revision_id)
```

Импортируйте `revision_view_view` из `views_pages`.

### Шаг I1.3. Шаблон страницы просмотра ревизии

Создайте `templates/revision_view.html`: заголовок страницы (например, «Ревизия #{{ revision.id }}» или «Просмотр ревизии: {{ revision.page_title }}»), блок с метаданными (страница — ссылка по `page_slug`, статус ревизии, автор, дата), затем **содержимое ревизии** (`revision.content`) в виде только для чтения (например, `<pre>` или блок с переносами строк). Внизу — ссылка «К странице» (на `page_view` по slug) и при `can_approve_reject` две формы: «Утвердить» (POST на `revision_approve`) и «Отклонить» (POST на `revision_reject` с необязательным полем `review_note`).

Пример:

```html
{% extends "base.html" %}

{% block title %}Ревизия #{{ revision.id }} · {{ revision.page_title }} · Вики{% endblock %}

{% block content %}
  <section class="card">
    <h1>Просмотр ревизии</h1>
    <p><a href="{{ url_for('page_view', slug=revision.page_slug) }}">{{ revision.page_title }}</a></p>
    <p class="muted">
      Статус: {{ revision.status }} · Автор: {{ revision.author_username }} · {{ revision.created_at }}
    </p>

    <div class="wiki-content">
      <pre>{{ revision.content or '' }}</pre>
    </div>

    {% if can_approve_reject %}
      <p><strong>Действия:</strong></p>
      <form method="post" action="{{ url_for('revision_approve', revision_id=revision.id) }}" class="inline-form">
        <button type="submit" class="btn btn-primary">Утвердить</button>
      </form>
      <form method="post" action="{{ url_for('revision_reject', revision_id=revision.id) }}" class="inline-form">
        <input type="text" name="review_note" placeholder="Заметка (необязательно)" size="24" />
        <button type="submit" class="btn btn-danger">Отклонить</button>
      </form>
    {% endif %}

    <p style="margin-top: 16px;">
      <a href="{{ url_for('page_view', slug=revision.page_slug) }}">К странице</a>
      ·
      <a href="{{ url_for('dashboard') }}">Дашборд</a>
    </p>
  </section>
{% endblock %}
```

---

## I2. Представления «Утвердить» и «Отклонить» в `views_pages.py`

Действия доступны только пользователям с ролью редактор или мастер (`is_editor()`). Ревизия должна существовать и иметь статус `pending`.

### Шаг I2.1. Утверждение ревизии

Добавьте представление, которое по id ревизии переводит её из `pending` в `approved` и записывает текущего пользователя в `reviewer_id`. После обновления — редирект на страницу (по slug) или на дашборд и flash «Ревизия утверждена».

Пример:

```python
def revision_approve_view(revision_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_editor():
        abort(403)

    conn = get_conn()
    rev = conn.execute(
        "SELECT id, page_id, author_id, status FROM revisions WHERE id = ?",
        (revision_id,),
    ).fetchone()
    if rev is None or rev["status"] != "pending":
        conn.close()
        abort(404)

    user = current_user()
    conn.execute(
        "UPDATE revisions SET status = 'approved', reviewer_id = ? WHERE id = ?",
        (user["id"], revision_id),
    )
    conn.commit()
    page = conn.execute("SELECT slug FROM pages WHERE id = ?", (rev["page_id"],)).fetchone()
    conn.close()

    flash("Ревизия утверждена.")
    if page:
        return redirect(url_for("page_view", slug=page["slug"]))
    return redirect(url_for("dashboard"))
```

### Шаг I2.2. Отклонение ревизии

Добавьте представление, которое по id ревизии переводит её из `pending` в `rejected`, записывает `reviewer_id` и необязательную заметку рецензента `review_note` из тела запроса (например, `request.form.get("review_note")`). Метод — POST. После обновления — редирект на страницу или дашборд и flash «Ревизия отклонена».

Пример:

```python
def revision_reject_view(revision_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_editor():
        abort(403)

    conn = get_conn()
    rev = conn.execute(
        "SELECT id, page_id, author_id, status FROM revisions WHERE id = ?",
        (revision_id,),
    ).fetchone()
    if rev is None or rev["status"] != "pending":
        conn.close()
        abort(404)

    user = current_user()
    review_note = (request.form.get("review_note") or "").strip()

    conn.execute(
        "UPDATE revisions SET status = 'rejected', reviewer_id = ?, review_note = ? WHERE id = ?",
        (user["id"], review_note or None, revision_id),
    )
    conn.commit()
    page = conn.execute("SELECT slug FROM pages WHERE id = ?", (rev["page_id"],)).fetchone()
    conn.close()

    flash("Ревизия отклонена.")
    if page:
        return redirect(url_for("page_view", slug=page["slug"]))
    return redirect(url_for("dashboard"))
```

---

## I3. Маршруты в `app.py`

### Шаг I3.1. Импорт представлений

В блоке импорта из `views_pages` добавьте `revision_view_view`, `revision_approve_view` и `revision_reject_view`:

```python
from views_pages import (
    ...
    revision_view_view,
    revision_approve_view,
    revision_reject_view,
)
```

### Шаг I3.2. Маршруты для ревизий

Добавьте маршруты рядом друг с другом (например, после маршрута `revision_submit`): GET просмотра ревизии, POST утверждения и POST отклонения.

```python
@app.get("/revisions/<int:revision_id>")
def revision_view(revision_id):
    return revision_view_view(revision_id)


@app.post("/revisions/<int:revision_id>/approve")
def revision_approve(revision_id):
    return revision_approve_view(revision_id)


@app.post("/revisions/<int:revision_id>/reject")
def revision_reject(revision_id):
    return revision_reject_view(revision_id)
```

(Если маршрут GET `/revisions/<int:revision_id>` вы уже добавили в шаге I1.2, убедитесь, что он находится рядом с этими маршрутами и не дублируется.)

---

## I4. Дашборд: «Очередь ожидающих» и «Мои правки на проверке» со ссылкой «Просмотр»

### Шаг I4.1. Импорт `is_editor` в `views_auth.py`

В начале файла `views_auth.py` добавьте `is_editor` в импорт из `auth_utils`:

```python
from auth_utils import is_logged_in, current_user, get_registration_open, is_editor
```

### Шаг I4.2. Данные для дашборда в `dashboard_view`

В одной сессии с БД (одно соединение) выполните: (1) запрос черновиков текущего пользователя (`draft_revisions`, как в Части H); (2) при `is_editor()` — запрос всех ревизий со статусом `pending` с присоединением страницы и автора (`pending_queue`); (3) запрос ревизий текущего пользователя со статусом `pending` с данными страницы (`my_pending`). Преобразуйте результаты в списки словарей. Закройте соединение и передайте в шаблон `draft_revisions`, `pending_queue` и `my_pending` (для не-редакторов `pending_queue` — пустой список).

Пример (вставьте в `dashboard_view()` после проверки пользователя, заменив существующий блок с `conn = get_conn()` и запросом черновиков):

```python
    conn = get_conn()
    rows = conn.execute(
        """
        SELECT r.id AS revision_id, r.page_id, r.content, r.created_at, p.slug, p.title
        FROM revisions r
        JOIN pages p ON r.page_id = p.id
        WHERE r.author_id = ? AND r.status = 'draft'
        ORDER BY r.created_at DESC
        """,
        (user["id"],),
    ).fetchall()
    draft_revisions = [dict(row) for row in rows]

    pending_queue = []
    if is_editor():
        pending_queue = [dict(row) for row in conn.execute(
            """
            SELECT r.id AS revision_id, r.page_id, r.created_at, p.slug, p.title, u.username AS author_username
            FROM revisions r
            JOIN pages p ON r.page_id = p.id
            LEFT JOIN users u ON r.author_id = u.id
            WHERE r.status = 'pending'
            ORDER BY r.created_at ASC
            """
        ).fetchall()]

    my_pending = [dict(row) for row in conn.execute(
        """
        SELECT r.id AS revision_id, r.page_id, r.created_at, p.slug, p.title
        FROM revisions r
        JOIN pages p ON r.page_id = p.id
        WHERE r.author_id = ? AND r.status = 'pending'
        ORDER BY r.created_at DESC
        """,
        (user["id"],),
    ).fetchall()]
    conn.close()

    return render_template("dashboard.html", user=user, draft_revisions=draft_revisions, pending_queue=pending_queue, my_pending=my_pending)
```

### Шаг I4.3. Шаблон дашборда: блок «Очередь ожидающих» со ссылкой «Просмотр»

В `templates/dashboard.html` после блока «Мои черновики» добавьте блок «Очередь ожидающих»: он отображается только редакторам и мастерам (проверка через `is_editor()` в шаблоне) и только если передан непустой список `pending_queue`. В таблице выведите: страница (ссылка по slug), автор ревизии, дата, действия. В колонке «Действия» — ссылка **«Просмотр»** на страницу просмотра ревизии (`url_for('revision_view', revision_id=rev.revision_id)`), форма «Утвердить» (POST на `revision_approve`) и форма «Отклонить» с необязательным полем заметки (POST на `revision_reject`).

Пример фрагмента:

```html
  {% if is_editor() and pending_queue %}
    <section class="card" style="margin-top: 16px;">
      <h2>Очередь ожидающих</h2>
      <p class="muted">Ревизии, ожидающие утверждения или отклонения.</p>
      <table class="table">
        <thead>
          <tr>
            <th>Страница</th>
            <th>Автор</th>
            <th>Дата</th>
            <th>Действия</th>
          </tr>
        </thead>
        <tbody>
          {% for rev in pending_queue %}
            <tr>
              <td><a href="{{ url_for('page_view', slug=rev.slug) }}">{{ rev.title }}</a></td>
              <td>{{ rev.author_username or '—' }}</td>
              <td>{{ rev.created_at[:10] if rev.created_at else '' }}</td>
              <td class="actions-cell">
                <a href="{{ url_for('revision_view', revision_id=rev.revision_id) }}" class="btn btn-sm">Просмотр</a>
                <form method="post" action="{{ url_for('revision_approve', revision_id=rev.revision_id) }}" class="inline-form">
                  <button type="submit" class="btn btn-sm btn-primary">Утвердить</button>
                </form>
                <form method="post" action="{{ url_for('revision_reject', revision_id=rev.revision_id) }}" class="inline-form">
                  <input type="text" name="review_note" placeholder="Заметка (необязательно)" size="20" />
                  <button type="submit" class="btn btn-sm btn-danger">Отклонить</button>
                </form>
              </td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    </section>
  {% endif %}
```

Убедитесь, что в шаблон передаётся переменная `pending_queue` (пустой список для не-редакторов).

### Шаг I4.4. Шаблон дашборда: блок «Мои правки на проверке» со ссылкой «Просмотр»

В `templates/dashboard.html` добавьте блок «Мои правки на проверке»: отображается, если передан непустой список `my_pending`. Выведите заголовок и таблицу: страница (ссылка по slug), дата ревизии, колонка «Действие» со ссылкой **«Просмотр»** на `url_for('revision_view', revision_id=rev.revision_id)`, чтобы автор мог открыть и увидеть содержимое отправленной на проверку ревизии.

Пример фрагмента:

```html
  {% if my_pending %}
    <section class="card" style="margin-top: 16px;">
      <h2>Мои правки на проверке</h2>
      <p class="muted">Ваши правки ожидают утверждения или отклонения редактором.</p>
      <table class="table">
        <thead>
          <tr>
            <th>Страница</th>
            <th>Дата</th>
            <th>Действие</th>
          </tr>
        </thead>
        <tbody>
          {% for rev in my_pending %}
            <tr>
              <td><a href="{{ url_for('page_view', slug=rev.slug) }}">{{ rev.title }}</a></td>
              <td>{{ rev.created_at[:10] if rev.created_at else '' }}</td>
              <td>
                <a href="{{ url_for('revision_view', revision_id=rev.revision_id) }}" class="btn btn-sm">Просмотр</a>
              </td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    </section>
  {% endif %}
```

В `dashboard_view()` всегда передавайте `my_pending` (пустой список, если ревизий в ожидании нет).

---

## Visible / testable (что можно проверить)

- **Редактор или мастер:** На дашборде отображается блок «Очередь ожидающих» со списком ревизий в статусе `pending`; у каждой строки есть ссылка «Просмотр», кнопки «Утвердить» и «Отклонить». По «Просмотр» открывается страница с текстом ревизии; на той же странице доступны «Утвердить» и «Отклонить». После утверждения контент страницы обновляется; после отклонения ревизия переходит в `rejected`.
- **Автор:** При наличии отправленных на проверку ревизий отображается блок «Мои правки на проверке» со ссылкой «Просмотр» у каждой строки; по ней автор видит тот самый текст, который отправил на проверку.

---

## Итог Части I

После выполнения части I:

- **Просмотр ревизии:** маршрут GET `/revisions/<int:revision_id>`, представление `revision_view_view(revision_id)` с проверкой прав (редактор — любая ревизия; автор — своя draft/pending или любая approved), шаблон `revision_view.html` с выводом содержимого ревизии и при `can_approve_reject` — формами «Утвердить» и «Отклонить».
- **Утверждение и отклонение:** представления `revision_approve_view` и `revision_reject_view` в `views_pages.py` (pending → approved с `reviewer_id` или pending → rejected с `reviewer_id` и необязательной `review_note`); маршруты POST `/revisions/<int:revision_id>/approve` и POST `/revisions/<int:revision_id>/reject` в `app.py`.
- **Дашборд:** в `views_auth.py` импорт `is_editor`, в `dashboard_view()` в одном соединении запрашиваются `draft_revisions`, при `is_editor()` — `pending_queue`, затем `my_pending`; в шаблон передаются все три списка. В `templates/dashboard.html`: блок «Очередь ожидающих» (для редакторов) с ссылкой «Просмотр», кнопками «Утвердить» и «Отклонить»; блок «Мои правки на проверке» с ссылкой «Просмотр» у каждой ревизии.

Цикл модерации завершён: черновик → отправка на проверку → просмотр текста ревизии → утверждение или отклонение редактором. В следующей части (J) — страница истории ревизий и откат к выбранной утверждённой ревизии; на ней можно ссылаться на тот же маршрут просмотра ревизии по id.
{% endraw %}
