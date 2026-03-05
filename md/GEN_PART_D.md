{% raw %}
# Часть D (переписанная): Админ‑страница пользователей и soft delete

В Частях A–C вы уже сделали:

- базовую систему пользователей и входа/выхода;
- роль мастера (`admin`) и проверку доступа;
- флаг «регистрация открыта/закрыта» и общий шаблон `base.html`.

В этой переписанной Части D мы **по шагам** добавим:

1. вывод `flash`‑сообщений и ссылку «Пользователи» в шапке;
2. страницу `/admin/users` со списком пользователей;
3. создание пользователя в админке;
4. **soft delete** (архивирование и восстановление);
5. полное удаление пользователя (hard delete).

После каждого шага есть отдельный блок «Проверка», чтобы вы могли убедиться, что всё работает, прежде чем двигаться дальше.

---

## D1. Подготовка: `flash` и место для сообщений в макете

Сначала подготовим основу: импортируем `flash` и добавим в базовый шаблон место, где будут показываться сообщения об успешных/ошибочных действиях.

### 1.1. Импорт `flash` в `app.py`

Вверху `app.py` у вас уже есть импорт из Flask. Добавьте туда `flash`:

```python
from flask import ..., flash
```

Форма может отличаться, главное — чтобы среди импортируемых имён появилось `flash`.

### 1.2. Вывод `flash`‑сообщений в `base.html`

В Части C вы уже создали общий шаблон `base.html`. Теперь добавим в нём блок, где будут выводиться сообщения.

Найдите в `base.html` место сразу после `<header>...</header>` и перед `<main>`. Добавьте это:

```html
  <body>
    <header>
      ...
    </header>

    {% with messages = get_flashed_messages() %}
      {% if messages %}
        <section class="flash-container">
          {% for message in messages %}
            <div class="flash-message">
              {{ message }}
            </div>
          {% endfor %}
        </section>
      {% endif %}
    {% endwith %}

    <main>
      {% block content %}{% endblock %}
    </main>
  </body>
```

> Стили `flash-container` и `flash-message` можно добавить в `styles.css` по желанию (рамка, фон, отступы и т.п.). Например:

```css
.flash-message {
  padding: 12px 16px;
  margin: 5px 0;
  border: 1px solid #ccc;
  border-radius: 4px;
  background-color: #f9f9f9;
  color: #333;
}
```

**Проверка D1**

1. В любом обработчике (например, во входе) временно добавьте `flash("Проверка flash")` перед `return`.
2. Откройте страницу — сообщение должно появиться над основным содержимым.
3. Убедитесь, что после перезагрузки страницы сообщение исчезает (flash показывается один раз).

---

## D2. Навигация: ссылка «Пользователи» для мастера

Теперь добавим понятную ссылку в шапке, которая будет вести на админ‑страницу пользователей. Это исправляет ситуацию, когда в Части C была «висящая» ссылка‑заглушка.

В `base.html` в блоке навигации для мастера замените ссылку «Пользователи» на реальный маршрут:

```html
        {% if is_admin() %}
          <a href="{{ url_for('admin_settings') }}">Настройки</a>
          <a href="{{ url_for('admin_users') }}">Пользователи</a>
        {% endif %}
```

> Пока маршрут `admin_users` ещё не создан, при клике вы увидите ошибку 404 — это нормально. После шага D3 ссылка заработает.

**Проверка D2**

1. Войдите под мастером.
2. Убедитесь, что в шапке есть ссылка «Пользователи».
3. Нажмите на неё — пока может быть 404; в следующем шаге мы добавим саму страницу.

---

## D3. Страница `/admin/users`: список пользователей

Теперь создадим саму админ‑страницу пользователей. Она будет:

- доступна только мастеру;
- показывать таблицу всех пользователей;
- использовать шаблон `admin_users.html`.

### 3.1. Маршрут `admin_users` в `app.py`

Добавьте в `app.py`:

```python
@app.get("/admin/users")
def admin_users():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_admin():
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

Здесь:

- гости перенаправляются на форму входа;
- не‑мастер получает 403;
- мастер видит список всех пользователей.

### 3.2. Шаблон `admin_users.html` (пока только список)

Создайте файл `templates/admin_users.html` со следующим содержимым:

```html
{% extends "base.html" %}

{% block title %}Пользователи · Админ{% endblock %}

{% block content %}
  <section class="card">
    <h1>Пользователи</h1>

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
            <!-- Кнопки действий добавим в шагах D5–D6 -->
          </td>
        </tr>
        {% endfor %}
      </tbody>
    </table>
  </section>
{% endblock %}
```

**Проверка D3**

1. Перезапустите приложение.
2. Войдите под мастером и откройте `/admin/users` (или кликните по ссылке «Пользователи» в шапке).
3. Убедитесь, что:
   - страница доступна только мастеру;
   - вы видите таблицу с пользователями (минимум мастер‑аккаунт).

---

## D4. Создание пользователя на странице `/admin/users`

Теперь добавим в эту же страницу простую форму создания пользователя и обработчик, который:

- проверяет, что запрос делает мастер;
- валидирует логин и пароль;
- создаёт пользователя через `create_user(...)`;
- показывает результат через `flash`.

### 4.1. Форма создания пользователя в `admin_users.html`

Дополните верхнюю часть содержимого `admin_users.html` (внутри `{% block content %}`) формой. В итоге блок будет выглядеть так:

```html
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

    <table class="table">...</table>
  </section>
{% endblock %}
```

Главное — чтобы форма отправляла `POST` на `url_for('admin_user_create')` и имела поля `username` и `password`.

### 4.2. Обработчик `admin_user_create` в `app.py`

Добавьте маршрут:

```python
@app.post("/admin/users/create")
def admin_user_create():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_admin():
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
        # В универсальном варианте создаём пользователей с ролью 'user'
        create_user(username, password, "user")
        flash(f"Пользователь {username} создан.")
    except sqlite3.IntegrityError:
        flash("Такой логин уже занят.")

    return redirect(url_for("admin_users"))
```

Здесь используется уже существующая функция `create_user(...)`.

**Проверка D4**

1. Откройте `/admin/users` под мастером.
2. Заполните форму «Создать пользователя» и отправьте.
3. Убедитесь, что:
   - при валидных данных создаётся новый пользователь с ролью `user`;
   - при конфликте логина появляется `flash`‑сообщение «Такой логин уже занят.»;
   - новый пользователь виден в таблице.

---

## D5. Архивирование и восстановление пользователя (soft delete)

Теперь реализуем **soft delete** — временное отключение учётной записи без удаления строки.

В таблице `users` уже есть поле `archived_at`. Интерпретация:

- `archived_at IS NULL` — пользователь активен;
- `archived_at NOT NULL` — пользователь считается заархивированным и **не должен** иметь возможность входа.

### 5.1. Обновление проверки входа (на всякий случай)

Убедитесь, что в обработчике входа `login()` (`@app.post(/login)`) у вас уже есть фильтр по `archived_at IS NULL`:

```python
...
user = conn.execute(
        "SELECT * FROM users WHERE username = ? AND archived_at IS NULL LIMIT 1",
        (username,),
    ).fetchone()
...

```

> Это гарантирует, что заархивированный пользователь больше не сможет войти.

### 5.2. Маршрут `admin_user_archive`

Добавьте импорт в `app.py`:
```python
from datetime import datetime
```

И добавьте этот новый маршрут в `app.py`:

```python
@app.post("/admin/users/<int:user_id>/archive")
def admin_user_archive(user_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_admin():
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

### 5.3. Маршрут `admin_user_restore`

Добавьте обратный маршрут:

```python
@app.post("/admin/users/<int:user_id>/restore")
def admin_user_restore(user_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_admin():
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

### 5.4. Кнопки «Архивировать» и «Восстановить» в `admin_users.html`

Теперь добавим реальные кнопки действий в колонку «Действия» шаблона `admin_users.html`.

Замените содержимое ячейки `actions-cell` так:

```html
          <td class="actions-cell">
            {% if u.archived_at %}
              <form method="post" action="{{ url_for('admin_user_restore', user_id=u.id) }}" class="inline-form">
                <button type="submit" class="btn btn-sm">Восстановить</button>
              </form>
            {% elif u.role != 'admin'  %}
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
          </td>
```

**Проверка D5**

1. Откройте `/admin/users` под мастером.
2. Нажмите «Архивировать» для обычного пользователя.
3. Убедитесь, что:
   - статус меняется на «В архиве»;
   - при попытке входа под этим логином вход не удаётся;
   - кнопка меняется на «Восстановить».
4. Нажмите «Восстановить» и проверьте, что пользователь снова может войти.

---

## D6. Полное удаление пользователя (hard delete)

Иногда нужно удалить учётную запись полностью (тестовые аккаунты, дубликаты и т.п.). Это **необратимая** операция, поэтому:

- доступна только мастеру;
- не должна позволять удалить мастер‑аккаунт.

### 6.1. Маршрут `admin_user_delete`

Добавьте в `app.py`:

```python
@app.post("/admin/users/<int:user_id>/delete")
def admin_user_delete(user_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_admin():
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

### 6.2. Кнопка «Удалить навсегда» в `admin_users.html`

Добавим кнопку удаления в колонку «Действия» рядом с архивированием/восстановлением. Финальный вид ячейки:

```html
          <td class="actions-cell">
            {% if u.archived_at %}
              ...
            {% elif u.role != 'admin'  %}
              ...
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
```

**Проверка D6**

1. Создайте тестового пользователя через форму на `/admin/users`.
2. Нажмите «Удалить навсегда» для этого пользователя и подтвердите действие в диалоге.
3. Убедитесь, что:
   - пользователь исчез из списка;
   - при попытке войти под его логином вход невозможен;
   - мастер‑аккаунт не имеет кнопки «Удалить навсегда».

---

## Итог Части D (переписанной)

После выполнения этой версии Части D у вас есть:

- страница `/admin/users`, доступная только мастеру и доступная через ссылку «Пользователи» в шапке;
- форма создания пользователей в админке с валидацией и `flash`‑сообщениями;
- реализация паттерна **soft delete** для пользователей через поле `archived_at`:
  - маршруты `/admin/users/<id>/archive` и `/admin/users/<id>/restore`,
  - соответствующие кнопки в таблице;
- маршрут полного удаления пользователя `/admin/users/<id>/delete` с защитой от удаления мастер‑аккаунта;
- единый вывод `flash`‑сообщений в `base.html`, который используется всеми действиями админ‑раздела.

Это завершает универсальный слой управления пользователями, на который можно дальше опираться в предметной логике вашего проекта.
{% endraw %}

