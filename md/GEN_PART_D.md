{% raw %}
# Часть D: Админ‑страница пользователей и soft delete

В Частях A–C вы:

- создали таблицу `users` и вспомогательные функции работы с БД;
- добавили регистрацию, вход/выход, сессию и мастер‑аккаунт;
- настроили флаг «регистрация открыта/закрыта» и общий базовый шаблон.

Теперь добавим **минимальный админ‑раздел управления пользователями** и на его примере разберём простой паттерн:

- **soft delete / архивирование** — временное отключение учётной записи без удаления строки из БД;
- **полное удаление пользователя** — аккуратное удаление строки из таблицы `users`.

На этом этапе мы **не трогаем никакие другие таблицы** (тикеты, сообщения и т.п.) — вся логика относится только к таблице `users`, которая является универсальной для разных учебных проектов.

---

## D1. Страница `/admin/users` и импорт `flash`

Для админ‑страницы понадобятся всплывающие сообщения (`flash`), чтобы показывать результат действий (создан, заархивирован, удалён и т.п.).

### 1. Импорт `flash` в начале `app.py`

Вверху `app.py` у вас уже есть строка импорта Flask. Расширьте её, добавив `flash`:

```python
from flask import ..., flash
```

Если у вас импорт разбит на несколько строк — главное, чтобы среди импортируемых из `flask` имён появился `flash`.

### 2. Маршрут списка пользователей `/admin/users`

Добавьте в `app.py` маршрут, который:

- разрешён только для вошедшего мастера (`role = 'admin'`);
- получает всех пользователей из таблицы `users`;
- передаёт их в шаблон `admin_users.html`.

```python
@app.get("/admin/users")
def admin_users():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_master():
        abort(403)

    conn = get_conn()
    users = conn.execute(
        """
        SELECT id, username, role, archived_at, created_at
        FROM users
        ORDER BY archived_at IS NULL DESC, username
        """
    ).fetchall()
    conn.close()

    return render_template("admin_users.html", users=users)
```

Функции `is_logged_in()` и `is_master()` вы добавляли ранее: они используют сессию и роль пользователя.

---

## D2. Шаблон `admin_users.html`: список и формы действий

Создадим HTML‑шаблон, который:

- наследует `base.html`;
- в верхней части содержит форму для создания нового пользователя;
- ниже показывает таблицу всех пользователей и кнопки действий (архивировать, восстановить, удалить навсегда).

Создайте файл `templates/admin_users.html` со следующим содержимым:

```html
{% extends "base.html" %}

{% block title %}Пользователи · Админ{% endblock %}

{% block content %}
  <section class="card">
    <h1>Пользователи</h1>

    <h2>Создать пользователя</h2>
    <form method="post" action="{{ url_for('admin_user_create') }}" class="form form-row">
      <div class="form-group">
        <label for="new_username">Логин</label>
        <input
          type="text"
          id="new_username"
          name="username"
          required
          minlength="2"
          placeholder="username"
        />
      </div>
      <div class="form-group">
        <label for="new_password">Пароль</label>
        <input
          type="password"
          id="new_password"
          name="password"
          required
          placeholder="••••••"
        />
      </div>
      <div class="form-group" style="align-self: flex-end;">
        <button type="submit" class="btn btn-primary">Создать</button>
      </div>
    </form>

    <p class="muted">
      Архивировать — пользователь не может войти, но его запись остаётся в таблице.
      Восстановить — снять архив, снова разрешить вход.
      Удалить навсегда — полностью удалить пользователя из таблицы (без возможности восстановления).
    </p>

    <table class="table">
      <thead>
        <tr>
          <th>ID</th>
          <th>Логин</th>
          <th>Роль</th>
          <th>Статус</th>
          <th>Создан</th>
          <th>Действия</th>
        </tr>
      </thead>
      <tbody>
        {% for u in users %}
        <tr>
          <td>{{ u.id }}</td>
          <td>{{ u.username }}</td>
          <td>{{ u.role }}</td>
          <td>
            {% if u.archived_at %}
              <span class="badge badge-deleted">В архиве</span>
            {% else %}
              Активен
            {% endif %}
          </td>
          <td>{{ u.created_at[:10] if u.created_at else '' }}</td>
          <td class="actions-cell">
            {% if u.archived_at %}
              <form method="post" action="{{ url_for('admin_user_restore', user_id=u.id) }}" class="inline-form">
                <button type="submit" class="btn btn-sm">Восстановить</button>
              </form>
            {% else %}
              <form method="post" action="{{ url_for('admin_user_archive', user_id=u.id) }}" class="inline-form">
                <button
                  type="submit"
                  class="btn btn-sm btn-danger"
                  onclick="return confirm('Заархивировать пользователя? Он не сможет войти.')"
                >
                  Архивировать
                </button>
              </form>
            {% endif %}

            {% if u.role != 'admin' %}
              <form method="post" action="{{ url_for('admin_user_delete', user_id=u.id) }}" class="inline-form">
                <button
                  type="submit"
                  class="btn btn-sm btn-danger"
                  onclick="return confirm('Удалить пользователя безвозвратно? Это действие нельзя отменить.')"
                >
                  Удалить навсегда
                </button>
              </form>
            {% endif %}
          </td>
        </tr>
        {% endfor %}
      </tbody>
    </table>
  </section>
{% endblock %}
```

Навигацию к этой странице (`/admin/users`) вы можете добавить в `base.html` там, где уже показываются ссылки для мастера.

---

## D3. Создание пользователя в админ‑разделе

Форма в `admin_users.html` отправляет POST‑запрос на `/admin/users/create`. Реализуем этот маршрут в `app.py`, используя функцию `create_user`, которую вы подготовили ранее.

### 1. Маршрут `admin_user_create`

Добавьте в `app.py`:

```python
@app.post("/admin/users/create")
def admin_user_create():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_master():
        abort(403)

    username = (request.form.get("username") or "").strip()
    password = request.form.get("password") or ""

    if not username or not password:
        flash("Введите логин и пароль.")
        return redirect(url_for("admin_users"))

    if len(username) < 2:
        flash("Логин слишком короткий.")
        return redirect(url_for("admin_users"))

    try:
        # В универсальном варианте можно создавать пользователей с ролью 'user'
        create_user(username, password, "user")
        flash(f"Пользователь {username} создан.")
    except sqlite3.IntegrityError:
        flash("Такой логин уже занят.")

    return redirect(url_for("admin_users"))
```

Здесь:

- проверяем права мастера;
- валидируем поля;
- создаём пользователя с ролью по умолчанию (`user`);
- показываем результат через `flash`.

---

## D4. Архивирование и восстановление пользователя (soft delete)

В таблице `users` уже есть поле `archived_at`. Используем его как **soft delete флаг**:

- если поле `archived_at` пустое (`NULL`) — пользователь активен;
- если заполнено — пользователь считается заархивированным и не должен иметь возможность входа в систему.

### 1. Маршрут `admin_user_archive`

Добавьте в `app.py`:

```python
@app.post("/admin/users/<int:user_id>/archive")
def admin_user_archive(user_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_master():
        abort(403)

    conn = get_conn()
    u = conn.execute("SELECT id FROM users WHERE id = ?", (user_id,)).fetchone()
    if u is None:
        conn.close()
        abort(404)

    now = datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%SZ")
    conn.execute("UPDATE users SET archived_at = ? WHERE id = ?", (now, user_id))
    conn.commit()
    conn.close()

    flash("Пользователь заархивирован. Он больше не сможет войти.")
    return redirect(url_for("admin_users"))
```

> Напоминание: в запросах, которые ищут пользователя для входа (`/login`), мы уже добавляли условие `archived_at IS NULL`, поэтому заархивированный пользователь автоматически перестаёт проходить проверку.

### 2. Маршрут `admin_user_restore`

Теперь сделаем обратную операцию — **снять архив**:

```python
@app.post("/admin/users/<int:user_id>/restore")
def admin_user_restore(user_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_master():
        abort(403)

    conn = get_conn()
    u = conn.execute("SELECT id FROM users WHERE id = ?", (user_id,)).fetchone()
    if u is None:
        conn.close()
        abort(404)

    conn.execute("UPDATE users SET archived_at = NULL WHERE id = ?", (user_id,))
    conn.commit()
    conn.close()

    flash("Пользователь восстановлен.")
    return redirect(url_for("admin_users"))
```

Так мы демонстрируем базовый паттерн **soft delete** на практике:

- «удалить» — значит пометить запись временной меткой;
- «вернуть» — очистить метку;
- реальный SQL‑запрос на выборку сам по себе не меняется — меняется только условие в `WHERE`.

---

## D5. Полное удаление пользователя (hard delete)

Иногда нужно удалить учётную запись полностью:

- тестовые аккаунты;
- дубликаты;
- пользователи, которых не требуется хранить в системе.

Здесь важно:

- ограничить действие только мастером;
- не позволять удалить мастер‑аккаунт;
- явно предупредить об окончательности операции.

### 1. Маршрут `admin_user_delete`

Добавьте в `app.py`:

```python
@app.post("/admin/users/<int:user_id>/delete")
def admin_user_delete(user_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_master():
        abort(403)

    conn = get_conn()
    u = conn.execute("SELECT id, username, role FROM users WHERE id = ?", (user_id,)).fetchone()
    if u is None:
        conn.close()
        abort(404)

    if u["role"] == "admin":
        # Мастер‑аккаунт удалять нельзя
        conn.close()
        flash("Нельзя удалить мастер‑аккаунт.")
        return redirect(url_for("admin_users"))

    conn.execute("DELETE FROM users WHERE id = ?", (user_id,))
    conn.commit()
    conn.close()

    flash(f"Пользователь {u['username']} удалён безвозвратно.")
    return redirect(url_for("admin_users"))
```

Это **жёсткое удаление** (hard delete):

- строка в таблице `users` реально исчезает;
- операция необратима;
- ограничена только мастером и не применяется к самому мастеру.

В более сложных проектах перед таким удалением нужно отдельно продумывать поведение связанного контента (записей в других таблицах). На уровне универсальной инструкции в этой части мы сознательно ограничиваемся только таблицей `users`.

---

## Итог Части D

После выполнения этой части у вас есть:

- страница `/admin/users`, доступная только мастеру;
- форма создания пользователей, использующая `create_user(...)` и `flash`‑сообщения;
- реализация паттерна **soft delete** для пользователей через поле `archived_at`:
  - маршруты `/admin/users/<id>/archive` и `/admin/users/<id>/restore`;
- маршрут полного удаления пользователя `/admin/users/<id>/delete` с защитой от удаления мастер‑аккаунта.

Это завершает универсальный слой управления пользователями. В следующих частях можно переходить к предметной логике конкретного проекта, опираясь на уже готовые паттерны: роли, админ‑страницы, soft delete и аккуратное удаление данных.
{% endraw %}

