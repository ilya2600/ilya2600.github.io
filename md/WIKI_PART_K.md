{% raw %}
# Часть K: Блокировка и удаление страниц, повышение и понижение роли пользователя

После Части J доступны история ревизий и откат. В этой части вы добавляете:

1. **Управление страницей (только мастер):** на странице просмотра страницы для пользователя с ролью **admin** (мастер) отображаются кнопки/ссылки **Заблокировать** / **Разблокировать** и **Удалить**. Блокировка уже учитывается при редактировании (Часть H): заблокированную страницу не может править автор. Удаление — с подтверждением; после удаления редирект (например, на список страниц); удаляются страница и все её ревизии.
2. **Управление ролями пользователей:** на странице списка пользователей (админка) отображается роль каждого пользователя; для **author** — действие **«Повысить до редактора»** (author → editor); для **editor** — **«Понизить до автора»** (editor → author); для **admin** действий по смене роли нет. В данной части добавляются только повышение и понижение роли; архивация и безвозвратное удаление пользователей в описание части K не входят (если они уже есть в проекте, их можно оставить).
3. **Дашборд и видимость:** на дашборде в одном месте представлены блоки «Мои черновики», «Мои правки на проверке», «Очередь ожидающих» (для редакторов и мастеров). По желанию добавьте на дашборд ссылку **«Новая страница»** на создание страницы. Список страниц и главная показывают только страницы, у которых есть хотя бы одна ревизия со статусом `approved` (как в Части G).

**Видимо:** мастер на странице видит «Заблокировать» / «Разблокировать» и «Удалить»; после блокировки автор не может редактировать; после удаления страница исчезает, редирект на список. В списке пользователей у автора — «Повысить до редактора», у редактора — «Понизить до автора»; после смены роли она видна в списке и в поведении (например, редактор видит «Очередь ожидающих» на дашборде). Дашборд содержит все нужные блоки; при желании — ссылку на новую страницу.

---

## K1. Блокировка, разблокировка и удаление страницы (только мастер)

Действия доступны только пользователю с ролью **admin** (`is_admin()`). Поле `is_locked` в таблице `pages` уже есть (Часть G/H); блокировка уже проверяется в форме редактирования.

### Шаг K1.1. Представления в `views_pages.py`

Добавьте три представления (или два: «переключить блокировку» и «удалить»).

**Заблокировать:** POST, по `slug` страницы выставить `is_locked = 1` для этой страницы. Проверка: `is_admin()`, страница существует. После обновления — редирект на страницу просмотра и flash «Страница заблокирована».

**Разблокировать:** POST, по `slug` выставить `is_locked = 0`. Аналогичные проверки и редирект с flash «Страница разблокирована».

**Удалить:** POST, по `slug` удалить все ревизии этой страницы, затем удалить саму страницу. Проверка: `is_admin()`, страница существует. После удаления — редирект на список страниц (или главную) и flash «Страница удалена».

Примеры:

```python
def page_lock_view(slug: str):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_admin():
        abort(403)

    conn = get_conn()
    page = conn.execute("SELECT id, slug FROM pages WHERE slug = ?", (slug,)).fetchone()
    if page is None:
        conn.close()
        abort(404)
    conn.execute("UPDATE pages SET is_locked = 1 WHERE id = ?", (page["id"],))
    conn.commit()
    conn.close()
    flash("Страница заблокирована.")
    return redirect(url_for("page_view", slug=slug))


def page_unlock_view(slug: str):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_admin():
        abort(403)

    conn = get_conn()
    page = conn.execute("SELECT id, slug FROM pages WHERE slug = ?", (slug,)).fetchone()
    if page is None:
        conn.close()
        abort(404)
    conn.execute("UPDATE pages SET is_locked = 0 WHERE id = ?", (page["id"],))
    conn.commit()
    conn.close()
    flash("Страница разблокирована.")
    return redirect(url_for("page_view", slug=slug))


def page_delete_view(slug: str):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_admin():
        abort(403)

    conn = get_conn()
    page = conn.execute("SELECT id, slug FROM pages WHERE slug = ?", (slug,)).fetchone()
    if page is None:
        conn.close()
        abort(404)
    conn.execute("DELETE FROM revisions WHERE page_id = ?", (page["id"],))
    conn.execute("DELETE FROM pages WHERE id = ?", (page["id"],))
    conn.commit()
    conn.close()
    flash("Страница удалена.")
    return redirect(url_for("page_list"))
```

В начале файла добавьте импорт `is_admin` из `auth_utils`, если его ещё нет.

### Шаг K1.2. Маршруты в `app.py`

Добавьте маршруты POST **до** маршрута `GET /pages/<slug>` (например, рядом с другими маршрутами страницы):

```python
@app.post("/pages/<slug>/lock")
def page_lock(slug):
    return page_lock_view(slug)


@app.post("/pages/<slug>/unlock")
def page_unlock(slug):
    return page_unlock_view(slug)


@app.post("/pages/<slug>/delete")
def page_delete(slug):
    return page_delete_view(slug)
```

Импортируйте `page_lock_view`, `page_unlock_view`, `page_delete_view` из `views_pages`.

### Шаг K1.3. Кнопки на странице просмотра страницы

В `templates/page_view.html` для мастера (`is_admin()`) добавьте блок с действиями: при `page.is_locked` — форма с кнопкой «Разблокировать» (POST на `page_unlock`), иначе — форма «Заблокировать» (POST на `page_lock`); отдельно форма «Удалить» (POST на `page_delete`) с подтверждением через `onclick="return confirm('…')"`.

Пример фрагмента (разместите после блока с «Редактировать» / «История» / «Откат», перед ссылками «К списку страниц»):

```html
    {% if is_admin() %}
      <p class="actions-cell">
        {% if page.is_locked %}
          <form method="post" action="{{ url_for('page_unlock', slug=page.slug) }}" class="inline-form">
            <button type="submit" class="btn btn-sm">Разблокировать</button>
          </form>
        {% else %}
          <form method="post" action="{{ url_for('page_lock', slug=page.slug) }}" class="inline-form">
            <button type="submit" class="btn btn-sm">Заблокировать</button>
          </form>
        {% endif %}
        <form method="post" action="{{ url_for('page_delete', slug=page.slug) }}" class="inline-form">
          <button type="submit" class="btn btn-sm btn-danger"
            onclick="return confirm('Удалить страницу и все ревизии безвозвратно?')">
            Удалить
          </button>
        </form>
      </p>
    {% endif %}
```

Убедитесь, что в контексте шаблона доступна функция `is_admin` (она передаётся через `context_processor` в `app.py`).

---

## K2. Повышение и понижение роли пользователя

На странице «Пользователи» (админка) список уже может содержать колонку «Роль». В этой части добавляются действия: **Повысить до редактора** (только для пользователей с ролью `author`) и **Понизить до автора** (только для пользователей с ролью `editor`). Для пользователя с ролью `admin` действий по смене роли не показывать (мастера не понижают).

### Шаг K2.1. Представления в `views_admin.py`

Добавьте два представления.

**Повысить до редактора:** POST, параметр `user_id`. Проверка: текущий пользователь — `is_admin()`, целевой пользователь существует и имеет роль `author`. Обновить роль на `editor`. Редирект на список пользователей и flash «Пользователь повышен до редактора».

**Понизить до автора:** POST, параметр `user_id`. Проверка: текущий пользователь — `is_admin()`, целевой пользователь существует и имеет роль `editor`. Обновить роль на `author`. Редирект на список пользователей и flash «Пользователь понижен до автора».

Примеры:

```python
def admin_user_promote_view(user_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_admin():
        abort(403)

    conn = get_conn()
    u = conn.execute("SELECT id, username, role FROM users WHERE id = ? AND archived_at IS NULL", (user_id,)).fetchone()
    if u is None or u["role"] != "author":
        conn.close()
        abort(404)
    conn.execute("UPDATE users SET role = 'editor' WHERE id = ?", (user_id,))
    conn.commit()
    conn.close()
    flash(f"Пользователь {u['username']} повышен до редактора.")
    return redirect(url_for("admin_users"))


def admin_user_demote_view(user_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_admin():
        abort(403)

    conn = get_conn()
    u = conn.execute("SELECT id, username, role FROM users WHERE id = ? AND archived_at IS NULL", (user_id,)).fetchone()
    if u is None or u["role"] != "editor":
        conn.close()
        abort(404)
    conn.execute("UPDATE users SET role = 'author' WHERE id = ?", (user_id,))
    conn.commit()
    conn.close()
    flash(f"Пользователь {u['username']} понижен до автора.")
    return redirect(url_for("admin_users"))
```

### Шаг K2.2. Маршруты в `app.py`

Добавьте маршруты POST (например, рядом с другими маршрутами админки пользователей):

```python
@app.post("/admin/users/<int:user_id>/promote")
def admin_user_promote(user_id):
    return admin_user_promote_view(user_id)


@app.post("/admin/users/<int:user_id>/demote")
def admin_user_demote(user_id):
    return admin_user_demote_view(user_id)
```

Импортируйте `admin_user_promote_view` и `admin_user_demote_view` из `views_admin`.

### Шаг K2.3. Кнопки в шаблоне списка пользователей

В `templates/admin_users.html` в колонке «Действия» (или рядом с существующими действиями) для активных пользователей (`archived_at` пустой): для роли **author** — форма с кнопкой «Повысить до редактора» (POST на `admin_user_promote`); для роли **editor** — форма с кнопкой «Понизить до автора» (POST на `admin_user_demote`); для роли **admin** — действий по смене роли не выводить.

Пример фрагмента для одной строки (адаптируйте под существующую разметку таблицы):

```html
{% if not u.archived_at %}
  {% if u.role == 'author' %}
    <form method="post" action="{{ url_for('admin_user_promote', user_id=u.id) }}" class="inline-form">
      <button type="submit" class="btn btn-sm btn-primary">Повысить до редактора</button>
    </form>
  {% elif u.role == 'editor' %}
    <form method="post" action="{{ url_for('admin_user_demote', user_id=u.id) }}" class="inline-form">
      <button type="submit" class="btn btn-sm">Понизить до автора</button>
    </form>
  {% endif %}
{% endif %}
```

Колонка «Роль» должна отображаться в таблице (например, «author», «editor», «admin» или подписи «Автор», «Редактор», «Мастер»).

---

## K3. Дашборд и видимость списка страниц

### Шаг K3.1. Дашборд

Убедитесь, что на дашборде в одном месте отображаются:

- **Мои черновики** — при непустом `draft_revisions` (как в Части I).
- **Мои правки на проверке** — при непустом `my_pending` (как в Части I).
- **Очередь ожидающих** — для редакторов и мастеров при непустом `pending_queue` (как в Части I).

По желанию добавьте в начало дашборда (или в блок с приветствием) ссылку **«Новая страница»** на `url_for('page_new')`, чтобы пользователь мог быстро перейти к созданию страницы.

Пример ссылки:

```html
<p><a href="{{ url_for('page_new') }}" class="btn btn-primary">Новая страница</a></p>
```

### Шаг K3.2. Список страниц и главная

В Части G список страниц строится только из страниц, у которых есть хотя бы одна ревизия со статусом `approved`. Если в вашем проекте это ещё не сделано, примените тот же критерий в `page_list_view()` (запрос с `INNER JOIN revisions ... AND r.status = 'approved'`) и при выводе списка на главной странице. Подробности — в Части G.

---

## Visible / testable (что можно проверить)

- **Мастер:** На странице просмотра страницы видны «Заблокировать» / «Разблокировать» и «Удалить». После блокировки автор не может редактировать эту страницу; после разблокировки — может. После нажатия «Удалить» и подтверждения страница и все её ревизии удаляются, выполняется редирект на список страниц.
- **Список пользователей:** У автора отображается «Повысить до редактора», у редактора — «Понизить до автора»; у мастера (admin) таких кнопок нет. После повышения пользователь с ролью editor видит на дашборде «Очередь ожидающих»; после понижения — не видит.
- **Дашборд:** Отображаются блоки «Мои черновики», «Мои правки на проверке», «Очередь ожидающих» (для редакторов); при добавлении — ссылка «Новая страница». Список страниц и главная показывают только страницы с утверждённой ревизией.

---

## Итог Части K

После выполнения части K:

- **Управление страницей (мастер):** представления `page_lock_view(slug)`, `page_unlock_view(slug)`, `page_delete_view(slug)` в `views_pages.py`; маршруты POST `/pages/<slug>/lock`, `/pages/<slug>/unlock`, `/pages/<slug>/delete`. На странице просмотра страницы при `is_admin()` отображаются кнопки Заблокировать/Разблокировать и Удалить (удаление с подтверждением); после удаления удаляются ревизии и страница, редирект на список страниц.
- **Управление ролями:** представления `admin_user_promote_view(user_id)` и `admin_user_demote_view(user_id)` в `views_admin.py` (author → editor, editor → author); маршруты POST `/admin/users/<int:user_id>/promote` и `/admin/users/<int:user_id>/demote`. В списке пользователей отображается роль; для author — «Повысить до редактора», для editor — «Понизить до автора», для admin — действий по смене роли нет.
- **Дашборд:** все блоки (Мои черновики, Мои правки на проверке, Очередь ожидающих) в одном месте; по желанию — ссылка «Новая страница». Список страниц и главная — только страницы с утверждённой ревизией (как в Части G).

В результате мастер может блокировать, разблокировать и удалять страницы; в админке доступны повышение и понижение роли пользователей; дашборд и видимость соответствуют описанному поведению.
{% endraw %}
