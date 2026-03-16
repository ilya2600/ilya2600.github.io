{% raw %}
# Часть H: Редактирование страницы, блокировка и отправка черновика

После Части G страницы создаются с первой ревизией: автор — черновик, редактор/мастер — утверждённая. В этой части вы добавляете:

1. **Редактирование страницы:** форма редактирования создаёт **новую ревизию**; автор → статус `pending`, редактор/мастер → `approved`. Заблокированные страницы не может править автор; редактор и мастер могут.
2. **Отправку черновика на проверку:** действие «Отправить на проверку» переводит ревизию из `draft` в `pending`. На дашборде — блок «Мои черновики» со списком черновиков и кнопкой «Отправить на проверку».

**Видимо:** ссылка «Редактировать» на странице, форма редактирования; при сохранении автором ревизия уходит в ожидание, редактором — сразу утверждённая. На заблокированной странице автор не видит редактирование или видит сообщение о блокировке; редактор правит. На дашборде — «Мои черновики» и «Отправить на проверку»; после отправки ревизия становится `pending`.

---

## H1. Функция `is_editor()` в `auth_utils.py`

Редактор и мастер могут править заблокированные страницы, утверждать и отклонять ревизии (в следующих частях). Для проверки «редактор или мастер» введите одну функцию.

### Шаг H1.1. Добавить `is_editor()`

Откройте `auth_utils.py` и рядом с `is_author_or_above()` добавьте:

```python
def is_editor():
    """Может ли текущий пользователь править заблокированные страницы и модерировать (роли editor, admin)."""
    u = current_user()
    return u is not None and u["role"] in ("editor", "admin")
```

Дальше в части H эта функция используется, чтобы разрешать редактирование заблокированной страницы только редактору и мастеру.

---

## H2. Контент для формы редактирования и право на редактирование в `views_pages.py`

Форма редактирования должна подставлять «текущий» контент: если есть утверждённая ревизия — её текст; если нет (например, только черновик) — последнюю ревизию страницы (черновик автора или любая последняя для редактора). Право на редактирование: пользователь с ролью author/editor/admin и либо страница не заблокирована, либо пользователь — редактор или мастер (`is_editor()`).

### Шаг H2.1. Хелпер: последняя ревизия страницы (любой статус)

В `views_pages.py` добавьте функцию, возвращающую последнюю по времени ревизию страницы (любой статус), чтобы подставлять контент в форму, когда утверждённой ревизии ещё нет:

```python
def get_latest_revision(conn, page_id: int):
    """Вернуть последнюю по времени ревизию страницы (любой статус) или None."""
    return conn.execute(
        """
        SELECT id, page_id, author_id, content, status, created_at
        FROM revisions
        WHERE page_id = ?
        ORDER BY created_at DESC
        LIMIT 1
        """,
        (page_id,),
    ).fetchone()
```

### Шаг H2.2. Хелпер: контент для подстановки в форму редактирования

Добавьте функцию, которая по `page_id` и текущему пользователю возвращает текст для поля «содержимое» в форме:

- если есть утверждённая ревизия — её `content`;
- иначе — контент последней ревизии (черновик или любая), если её может видеть пользователь (автор своей ревизии или редактор/мастер).

Пример реализации:

```python
def get_edit_content(conn, page_id: int, user: dict) -> str:
    """Контент для формы редактирования: последняя утверждённая ревизия или последняя видимая (напр. черновик)."""
    rev_approved = get_latest_approved_revision(conn, page_id)
    if rev_approved is not None:
        return rev_approved["content"] or ""
    rev_latest = get_latest_revision(conn, page_id)
    if rev_latest is None:
        return ""
    # Черновик/ожидание: видит автор этой ревизии или редактор/мастер
    if rev_latest["author_id"] == user["id"] or user["role"] in ("editor", "admin"):
        return rev_latest["content"] or ""
    return ""
```

### Шаг H2.3. Право на редактирование страницы

Править страницу может пользователь с ролью author, editor или admin (`is_author_or_above()`), при этом **если страница заблокирована** — только редактор или мастер (`is_editor()`). В коде представлений (edit/save) проверяйте: сначала загрузить страницу по slug, затем проверить `is_author_or_above()` и при `page["is_locked"]` проверить `is_editor()`; при неудаче — `abort(403)` или редирект с сообщением «Страница заблокирована».

---

## H3. Представления: форма редактирования (GET) и сохранение (POST) в `views_pages.py`

### Шаг H3.1. Импорт `is_editor`

В начале `views_pages.py` добавьте в импорт из `auth_utils` функцию `is_editor`:

```python
from auth_utils import is_logged_in, current_user, is_author_or_above, is_editor
```

### Шаг H3.2. Представление GET: форма редактирования

Добавьте представление, которое по slug страницы показывает форму редактирования с подставленным контентом. Проверки: пользователь залогинен и имеет право на редактирование (author/editor/admin; при заблокированной странице — только editor/admin). Если страницы нет — 404; если прав нет — 403.

Пример:

```python
def page_edit_view(slug: str):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_author_or_above():
        abort(403)

    page = get_page_by_slug(slug)
    if page is None:
        abort(404)

    if page["is_locked"] and not is_editor():
        flash("Страница заблокирована. Редактировать могут только редактор и мастер.")
        return redirect(url_for("page_view", slug=slug))

    conn = get_conn()
    content = get_edit_content(conn, page["id"], current_user())
    conn.close()

    return render_template("page_edit.html", page=page, content=content, error=None)
```

### Шаг H3.3. Представление POST: сохранение новой ревизии

Добавьте представление, которое обрабатывает отправку формы: та же проверка прав (включая блокировку); создаётся **новая** ревизия с переданным контентом; **автор** → статус `pending`, **редактор или мастер** → статус `approved`. После сохранения — редирект на просмотр страницы и flash-сообщение (например, «Изменения сохранены» или «Изменения отправлены на проверку»).

Пример:

```python
def page_save_view(slug: str):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_author_or_above():
        abort(403)

    page = get_page_by_slug(slug)
    if page is None:
        abort(404)

    if page["is_locked"] and not is_editor():
        flash("Страница заблокирована.")
        return redirect(url_for("page_view", slug=slug))

    content = request.form.get("content") or ""
    user = current_user()

    conn = get_conn()
    if user["role"] in ("editor", "admin"):
        status = "approved"
    else:
        status = "pending"

    conn.execute(
        """INSERT INTO revisions (page_id, author_id, content, status) VALUES (?, ?, ?, ?)""",
        (page["id"], user["id"], content, status),
    )
    conn.commit()
    conn.close()

    if status == "approved":
        flash("Изменения сохранены.")
    else:
        flash("Изменения отправлены на проверку.")
    return redirect(url_for("page_view", slug=slug))
```

---

## H4. Маршруты редактирования в `app.py`

### Шаг H4.1. Импорт представлений

В блоке импорта из `views_pages` добавьте `page_edit_view` и `page_save_view`:

```python
from views_pages import (
    page_list_view,
    page_new_view,
    page_create_view,
    page_view_view,
    page_edit_view,
    page_save_view,
)
```

### Шаг H4.2. Маршруты GET и POST для редактирования

Добавьте маршруты (после маршрута просмотра страницы):

```python
@app.get("/pages/<slug>/edit")
def page_edit(slug):
    return page_edit_view(slug)


@app.post("/pages/<slug>/edit")
def page_save(slug):
    return page_save_view(slug)
```

---

## H5. Шаблон формы редактирования и ссылка «Редактировать» на странице

### Шаг H5.1. Шаблон `page_edit.html`

Создайте файл `templates/page_edit.html`: заголовок страницы (например, «Редактировать: {{ page.title }}»), форма с полем `content` (textarea), кнопка «Сохранить», ссылка «К просмотру страницы» или «Отмена» на `page_view`. Метод формы — POST, действие — `url_for('page_save', slug=page.slug)`. При наличии переменной `error` выведите её над формой.

Пример:

```html
{% extends "base.html" %}

{% block title %}Редактировать: {{ page.title }} · Вики{% endblock %}

{% block content %}
  <section class="card">
    <h1>Редактировать: {{ page.title }}</h1>

    {% if error %}
      <p class="bad">{{ error }}</p>
    {% endif %}

    <form method="post" action="{{ url_for('page_save', slug=page.slug) }}" class="form">
      <div class="form-group">
        <label for="content">Содержимое</label>
        <textarea id="content" name="content" rows="12">{{ content }}</textarea>
      </div>
      <button type="submit" class="btn btn-primary">Сохранить</button>
    </form>

    <p style="margin-top: 16px;">
      <a href="{{ url_for('page_view', slug=page.slug) }}">К просмотру страницы</a>
      ·
      <a href="{{ url_for('page_list') }}">К списку страниц</a>
    </p>
  </section>
{% endblock %}
```

### Шаг H5.2. Ссылка «Редактировать» и блокировка в `page_view.html`

На странице просмотра ссылку «Редактировать» видят только пользователи с правом на редактирование. Если страница заблокирована и текущий пользователь — автор, ссылку не показывать (или показать текст «Страница заблокирована для редактирования»). Для этого в представлении `page_view_view` передайте в шаблон флаг, например `can_edit`: True, если `is_author_or_above()` и (страница не заблокирована или `is_editor()`); и при необходимости `page_locked` для сообщения.

**В `views_pages.py`** в функции `page_view_view` перед `return render_template(...)` вычислите и передайте:

```python
    can_edit = False
    if is_author_or_above():
        can_edit = not page.get("is_locked") or is_editor()
    return render_template("page_view.html", page=page, can_edit=can_edit)
```

**В `templates/page_view.html`** после блока с контентом и перед ссылкой «К списку страниц» добавьте:

```html
    {% if can_edit %}
      <p><a href="{{ url_for('page_edit', slug=page.slug) }}" class="btn btn-primary">Редактировать</a></p>
    {% elif page.is_locked and is_author_or_above() %}
      <p class="muted">Страница заблокирована для редактирования.</p>
    {% endif %}
```

Чтобы в шаблоне были доступны `is_author_or_above` и `is_editor`, их нужно передать в контекст приложения или уже иметь в нём (например, через `context_processor`). В Части F в контекст передаётся `current_user` и `is_admin`; добавьте в `app.py` в функцию `inject()` (или аналог) передачу `is_author_or_above` и `is_editor`:

```python
from auth_utils import (
    ...
    is_author_or_above,
    is_editor,
)
# в inject():
    "is_author_or_above": is_author_or_above,
    "is_editor": is_editor,
```

Тогда в `page_view.html` можно использовать `is_author_or_above()` и при необходимости `is_editor()`.

---

## H6. Дашборд: «Мои черновики» и отправка черновика на проверку

### Шаг H6.1. Список черновиков текущего пользователя в `views_auth.py`

На дашборде нужен блок «Мои черновики»: ревизии со статусом `draft`, у которых `author_id` равен id текущего пользователя, с привязкой к странице (slug, title) для ссылки. В `views_auth.py` в функции `dashboard_view()` после проверки залогина и получения `user` выполните запрос к БД: выбрать ревизии с `status = 'draft'` и `author_id = user['id']`, присоединить таблицу `pages` по `page_id`, чтобы получить `slug` и `title`. Передайте список в шаблон, например `draft_revisions`.

Пример запроса (в `dashboard_view`, при наличии `conn` или через `get_conn()`):

```python
    conn = get_conn()
    draft_revisions = conn.execute(
        """
        SELECT r.id AS revision_id, r.page_id, r.content, r.created_at, p.slug, p.title
        FROM revisions r
        JOIN pages p ON r.page_id = p.id
        WHERE r.author_id = ? AND r.status = 'draft'
        ORDER BY r.created_at DESC
        """,
        (user["id"],),
    ).fetchall()
    conn.close()
```

Передайте `draft_revisions` в `render_template("dashboard.html", user=user, draft_revisions=draft_revisions)`.

### Шаг H6.2. Действие «Отправить на проверку» в `views_pages.py`

Добавьте представление, которое по id ревизии переводит её из `draft` в `pending`. Разрешить действие может только автор этой ревизии или редактор/мастер. Метод — POST; после проверки прав и существования ревизии со статусом `draft` обновите `status` на `'pending'`, затем редирект на дашборд или на страницу с flash «Отправлено на проверку».

Пример (маршрут можно задать как `/revisions/<int:revision_id>/submit` или `/pages/<slug>/submit-draft`; ниже — по id ревизии):

```python
def revision_submit_view(revision_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_author_or_above():
        abort(403)

    conn = get_conn()
    rev = conn.execute(
        "SELECT id, page_id, author_id, status FROM revisions WHERE id = ?",
        (revision_id,),
    ).fetchone()
    if rev is None or rev["status"] != "draft":
        conn.close()
        abort(404)

    user = current_user()
    if rev["author_id"] != user["id"] and not is_editor():
        conn.close()
        abort(403)

    conn.execute("UPDATE revisions SET status = 'pending' WHERE id = ?", (revision_id,))
    conn.commit()
    page = conn.execute("SELECT slug FROM pages WHERE id = ?", (rev["page_id"],)).fetchone()
    conn.close()

    flash("Черновик отправлен на проверку.")
    slug = page["slug"] if page else None
    if slug:
        return redirect(url_for("page_view", slug=slug))
    return redirect(url_for("dashboard"))
```

### Шаг H6.3. Маршрут POST для отправки черновика в `app.py`

Добавьте импорт `revision_submit_view` из `views_pages` и маршрут, например:

```python
@app.post("/revisions/<int:revision_id>/submit")
def revision_submit(revision_id):
    return revision_submit_view(revision_id)
```

### Шаг H6.4. Шаблон дашборда: блок «Мои черновики»

Откройте `templates/dashboard.html`. Добавьте блок «Мои черновики»: если передан список `draft_revisions` и он не пуст, выведите заголовок «Мои черновики» и таблицу (или список) ревизий: ссылка на страницу (по `slug`), дата ревизии, кнопка «Отправить на проверку» — форма с методом POST и действием `url_for('revision_submit', revision_id=rev.revision_id)` (или аналог с вашим именем маршрута). Для каждой ревизии используйте `revision_id` из переданных данных.

Пример фрагмента (после приветствия и роли):

```html
  {% if draft_revisions %}
    <section class="card" style="margin-top: 16px;">
      <h2>Мои черновики</h2>
      <p class="muted">Черновики можно отправить на проверку редактору.</p>
      <table class="table">
        <thead>
          <tr>
            <th>Страница</th>
            <th>Дата</th>
            <th>Действие</th>
          </tr>
        </thead>
        <tbody>
          {% for rev in draft_revisions %}
            <tr>
              <td><a href="{{ url_for('page_view', slug=rev.slug) }}">{{ rev.title }}</a></td>
              <td>{{ rev.created_at[:10] if rev.created_at else '' }}</td>
              <td>
                <form method="post" action="{{ url_for('revision_submit', revision_id=rev.revision_id) }}" class="inline-form">
                  <button type="submit" class="btn btn-sm btn-primary">Отправить на проверку</button>
                </form>
              </td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    </section>
  {% endif %}
```

Убедитесь, что в `dashboard_view` в шаблон передаётся переменная `draft_revisions` (пустой список, если черновиков нет), чтобы при отсутствии черновиков блок не выводился или не падал.

---

## Visible / testable (что можно проверить)

- На странице просмотра при наличии прав отображается ссылка «Редактировать»; при открытии формы подставлен текущий контент (утверждённая ревизия или черновик).
- Автор сохраняет правку → ревизия создаётся со статусом `pending`; flash «Изменения отправлены на проверку»; контент на странице пока не меняется (до утверждения в Части I).
- Редактор/мастер сохраняет правку → ревизия `approved`; flash «Изменения сохранены»; контент на странице обновляется.
- Заблокированная страница: автор не видит «Редактировать» или видит сообщение о блокировке; редактор/мастер видит ссылку и может редактировать.
- Дашборд: при наличии черновиков отображается блок «Мои черновики» со списком и кнопкой «Отправить на проверку»; после отправки ревизия переходит в `pending` (в Части I будет видна в «Ожидают проверки» / Pending queue).

---

## Итог Части H

После выполнения части H:

- **`auth_utils.py`:** функция `is_editor()` (роли editor, admin).
- **`views_pages.py`:** хелперы `get_latest_revision(conn, page_id)`, `get_edit_content(conn, page_id, user)`; представления `page_edit_view(slug)`, `page_save_view(slug)` (проверка блокировки и роли; новая ревизия: автор → `pending`, редактор/мастер → `approved`); представление `revision_submit_view(revision_id)` (draft → pending). В `page_view_view` передаётся `can_edit` в шаблон.
- **`views_auth.py`:** в `dashboard_view()` запрашиваются черновики текущего пользователя (`draft_revisions`) и передаются в шаблон дашборда.
- **`app.py`:** маршруты GET/POST `/pages/<slug>/edit`, POST `/revisions/<int:revision_id>/submit`; в контекст шаблонов добавлены `is_author_or_above` и `is_editor`.
- **Шаблоны:** `page_edit.html` (форма редактирования); в `page_view.html` — ссылка «Редактировать» при `can_edit` и сообщение о блокировке для автора; в `dashboard.html` — блок «Мои черновики» с кнопкой «Отправить на проверку».

Редактирование создаёт новую ревизию с учётом роли и блокировки; черновик можно отправить на проверку с дашборда. В следующей части — очередь ожидающих ревизий и утверждение/отклонение редактором.
{% endraw %}
