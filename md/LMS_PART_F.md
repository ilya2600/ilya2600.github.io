{% raw %}
# Часть F: Ролевой дашборд и классы — первая предметная сущность

В Частях A–E вы разнесли приложение по модулям:

- `app.py` — создание Flask‑приложения и тонкие маршруты;
- `db.py` — работа с базой (`init_db`, `get_conn` и т.п.);
- `auth_utils.py` — пользователи и права (`current_user`, `is_admin`, `is_logged_in`, `get_registration_open`);
- `views_auth.py` — логика входа, регистрации и дашборда;
- `views_admin.py` — логика админ‑страниц;
- шаблоны в `templates/`.

В этой части вы добавляете **предметную логику ЛМС**:

- ролевой дашборд с разным контентом для учеников, учителей, менеджеров и мастера и простой статистикой по ролям;
- хелперы `is_manager()` и `is_master()` в `auth_utils.py`;
- таблицу `classes` в базе;
- модуль `views_classes.py` с логикой списка и создания классов;
- маршруты `/classes` и `/classes/new` в `app.py`;
- шаблоны дашборда, списка классов и формы создания класса; обновлённую навигацию в `base.html`.

Домашние задания, занятия и привязка учеников к классам остаются на следующие части.

---

## F1. Таблица `classes` в `db.py`

Схема БД хранится в `init_db()`. Таблица `classes` — первая предметная сущность ЛМС (учебная группа).

### Шаг F1.1. Добавляем таблицу `classes`

Откройте `db.py` и найдите функцию `init_db()`. Сразу **после** блока `CREATE TABLE IF NOT EXISTS users (...)` добавьте создание таблицы `classes`:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS classes (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        )
    """)
```

В итоге `init_db()` создаёт таблицы `settings`, `users`, `classes`, затем выполняет `conn.commit()` и `conn.close()`.

Для ЛМС в таблице `users` используются роли `student`, `teacher`, `manager`, `admin`. Замените проверку ролей на перечисление, подходящее для ЛМС, чтобы дашборд и доступ к классам работали по ролям:

```python
  conn.execute("""
        CREATE TABLE IF NOT EXISTS users (
            ...
            role TEXT NOT NULL CHECK (role IN ('student', 'teacher', 'manager', 'admin')),
            ...
        )
    """)
```

**Важно:** Команда `CREATE TABLE IF NOT EXISTS` не изменит существующую таблицу. Если у вас уже есть база данных со старой таблицей `users` (`role IN ('user', 'admin')`), вы должны удалить её (`database.db`).

А также, очень важно закоменнтировать вызов функции `insert_test_user()` если она у вас вызывается, так как там по старому создается `user` роль, которая не существует.

---

## F2. Хелперы `is_manager` и `is_master` в `auth_utils.py`

В навигации и при проверке доступа к классам нужны две функции: «текущий пользователь — менеджер» и «текущий пользователь — мастер (администратор)».

### Шаг F2.1. Функции в `auth_utils.py`

Откройте `auth_utils.py`. Рядом с `is_admin()` добавьте:

```python
def is_manager():
    u = current_user()
    return u is not None and u["role"] == "manager"


def is_master():
    u = current_user()
    return u is not None and u["role"] == "admin"
```

Поведение: для залогиненного пользователя проверяется роль; в ЛМС «мастер» — это роль `admin`.

### Шаг F2.2. Доступ в шаблонах: обновляем `app.py` context_processor

Чтобы в шаблонах можно было вызывать `is_manager()` и `is_master()`, их нужно передать в контекст. Откройте `app.py`, найдите импорт из `auth_utils` и добавьте в него `is_manager` и `is_master`:

```python
from auth_utils import (
    ...
    is_manager,
    is_master,
)
```

Затем найдите функцию `inject()` (context_processor) и добавьте в возвращаемый словарь:

```python
@app.context_processor
def inject():
    return {
        ...
        "is_admin": ...,
        "is_manager": is_manager,
        "is_master": is_master,
        ...
    }
```

В шаблонах будут доступны `is_manager()`, `is_master()` и при необходимости `get_registration_open()`.

Самостоятельная регистрация должна создавать пользователей только с ролью **student**. В файле `views_auth.py`, в функции `register_view()`, замените вставку, использующую роль `'user'`, на `'student'`:

```python
conn.execute(
    "INSERT INTO users (username, password, role) VALUES (?, ?, 'student')",
    (username, generate_password_hash(password)),
)
```


---

## F3. Ролевой дашборд в `views_auth.py`

Дашборд должен показывать разный контент в зависимости от роли и простую статистику по пользователям (ученики, учителя, менеджеры).

### Шаг F3.1. Обновляем `dashboard_view()`

Откройте `views_auth.py`. В начале файла убедитесь, что импортирован `get_conn` из `db` (если ещё не импортирован). Найдите функцию `dashboard_view()` и замените её тело на:

```python
def dashboard_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    if user is None:
        session.clear()
        return redirect(url_for("login_form"))

    conn = get_conn()
    total_students = conn.execute(
        "SELECT COUNT(*) AS c FROM users WHERE role = 'student' AND archived_at IS NULL"
    ).fetchone()["c"]
    total_teachers = conn.execute(
        "SELECT COUNT(*) AS c FROM users WHERE role = 'teacher' AND archived_at IS NULL"
    ).fetchone()["c"]
    total_managers = conn.execute(
        "SELECT COUNT(*) AS c FROM users WHERE role = 'manager' AND archived_at IS NULL"
    ).fetchone()["c"]
    conn.close()

    return render_template(
        "dashboard.html",
        user=user,
        total_students=total_students,
        total_teachers=total_teachers,
        total_managers=total_managers,
    )
```

Логика: проверка входа, получение текущего пользователя, подсчёт активных пользователей по ролям из таблицы `users`, передача всех данных в шаблон. Вся реализация остаётся в `views_auth.py`; маршрут в `app.py` по‑прежнему только вызывает `dashboard_view()`.

---

## F4. Модуль `views_classes.py` — список и создание классов

Вся логика страниц «Список классов» и «Новый класс» выносится в отдельный модуль.

### Шаг F4.1. Создаём файл `views_classes.py`

В той же папке, где лежат `app.py`, `views_auth.py`, `views_admin.py`, создайте файл `views_classes.py`.

### Шаг F4.2. Импорты

В начало `views_classes.py` вставьте:

```python
from flask import render_template, redirect, url_for, request, abort

from db import get_conn
from auth_utils import is_logged_in, is_manager, is_master
```

### Шаг F4.3. Проверка доступа к управлению классами

Управлять классами (список и создание) могут только менеджер и мастер. Добавьте хелпер:

```python
def can_manage_classes():
    """Может ли текущий пользователь просматривать и создавать классы."""
    return is_logged_in() and (is_manager() or is_master())
```

### Шаг F4.4. View‑функции списка и создания класса

В том же файле добавьте три функции (без декораторов маршрутов):

```python
def class_list_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not can_manage_classes():
        abort(403)

    conn = get_conn()
    classes = conn.execute(
        "SELECT id, name, created_at FROM classes ORDER BY name"
    ).fetchall()
    conn.close()

    return render_template("class_list.html", classes=classes)


def class_new_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not can_manage_classes():
        abort(403)

    return render_template("class_new.html", error=None)


def class_create_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not can_manage_classes():
        abort(403)

    name = (request.form.get("name") or "").strip()
    if not name:
        return render_template("class_new.html", error="Введите название класса.")

    conn = get_conn()
    conn.execute("INSERT INTO classes (name) VALUES (?)", (name,))
    conn.commit()
    conn.close()

    return redirect(url_for("class_list"))
```

Импорт `flash` можно добавить при желании для сообщений после создания класса; для минимального варианта достаточно редиректа на список.

---

## F5. Маршруты классов в `app.py`

Маршруты только делегируют вызов в `views_classes`.

### Шаг F5.1. Импорт view‑функций

В `app.py` в блок импортов из view‑модулей добавьте:

```python
from views_classes import (
    class_list_view,
    class_new_view,
    class_create_view,
)
```

### Шаг F5.2. Объявление маршрутов

Ниже маршрутов админки (например, после `admin_user_delete`) добавьте:

```python
@app.get("/classes")
def class_list():
    return class_list_view()


@app.get("/classes/new")
def class_new():
    return class_new_view()


@app.post("/classes/new")
def class_create():
    return class_create_view()
```

Вся проверка прав и работа с БД остаётся в `views_classes.py`.

---

## F6. Шаблон дашборда `dashboard.html`

Дашборд показывает разный блок в зависимости от роли и статистику по пользователям.

### Шаг F6.1. Создаём или заменяем `templates/dashboard.html`

Содержимое файла `templates/dashboard.html`:

```html
{% extends "base.html" %}

{% block title %}Дашборд · Мини‑ЛМС{% endblock %}

{% block content %}
  <section class="card">
    <h1>Дашборд</h1>
    <p>Вы вошли как <strong>{{ user.username }}</strong> (роль: {{ user.role }}).</p>

    {% if user.role == 'student' %}
      <h2>Для ученика</h2>
      <p>
        Здесь позже появятся ваши занятия, домашние задания и прогресс.
        Пока что вы можете войти под менеджером или учителем, чтобы настроить систему.
      </p>

    {% elif user.role == 'teacher' %}
      <h2>Для учителя</h2>
      <p>
        Здесь позже появятся ваши занятия и список домашних заданий для проверки.
      </p>

    {% elif user.role == 'manager' %}
      <h2>Для менеджера</h2>
      <p>
        Вы можете создавать учебные классы, планировать занятия и смотреть общую статистику.
      </p>
      <ul>
        <li><a href="{{ url_for('class_list') }}">Список классов</a></li>
      </ul>
      <h3>Краткая статистика по пользователям</h3>
      <ul>
        <li>Учеников: {{ total_students }}</li>
        <li>Учителей: {{ total_teachers }}</li>
        <li>Менеджеров: {{ total_managers }}</li>
      </ul>

    {% elif user.role == 'admin' %}
      <h2>Для мастера</h2>
      <p>
        Вы можете управлять пользователями и настраивать систему. В дальнейшем у мастера
        будут доступны дополнительные отчёты и инструменты.
      </p>
      <ul>
        <li><a href="{{ url_for('admin_users') }}">Пользователи</a></li>
        <li><a href="{{ url_for('admin_settings') }}">Настройки регистрации</a></li>
        <li><a href="{{ url_for('class_list') }}">Список классов</a></li>
      </ul>
      <h3>Краткая статистика по пользователям</h3>
      <ul>
        <li>Учеников: {{ total_students }}</li>
        <li>Учителей: {{ total_teachers }}</li>
        <li>Менеджеров: {{ total_managers }}</li>
      </ul>

    {% else %}
      <p>Для этой роли пока нет отдельного дашборда.</p>
    {% endif %}
  </section>
{% endblock %}
```

Шаблон ожидает переменные `user`, `total_students`, `total_teachers`, `total_managers`, которые передаёт обновлённый `dashboard_view()` из `views_auth.py`.

---

## F7. Навигация в `base.html` с учётом ролей ЛМС

Чтобы у менеджера и мастера в меню была ссылка «Классы», а у мастера — также «Настройки» и «Пользователи», обновите навигацию в базовом шаблоне.

### Шаг F7.1. Блок навигации в `templates/base.html`

Откройте `templates/base.html`, найдите блок навигации (например, `<nav>...</nav>`) и приведите его к виду с учётом ролей. Пример:

```html
<nav>
  ...

  {% if current_user() %}
    ...

    {% if is_manager() or is_master() %}
      <a href="{{ url_for('class_list') }}">Классы</a>
    {% endif %}

    {% if is_admin() %}
      ...
    {% endif %}

    ...
  {% else %}
    ...
  {% endif %}
</nav>
```

Здесь используются функции, доступные в шаблонах через context_processor: `current_user`, `is_manager`, `is_master`, `is_admin`, `get_registration_open`. Имена маршрутов (`login_form`, `register_form`, `admin_settings`, `admin_users`, `class_list`, `dashboard`, `logout`, `home`) должны совпадать с теми, что объявлены в `app.py`.

---

## F8. Шаблоны списка и создания класса

### Шаг F8.1. Список классов `templates/class_list.html`

Создайте файл `templates/class_list.html`:

```html
{% extends "base.html" %}

{% block title %}Классы · Мини‑ЛМС{% endblock %}

{% block content %}
  <section class="card">
    <h1>Классы</h1>

    <p>
      <a href="{{ url_for('class_new') }}" class="btn btn-primary">Новый класс</a>
    </p>

    {% if not classes %}
      <p class="muted">Пока нет ни одного класса.</p>
    {% else %}
      <table class="table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Название</th>
            <th>Создан</th>
          </tr>
        </thead>
        <tbody>
          {% for cls in classes %}
            <tr>
              <td>{{ cls.id }}</td>
              <td>{{ cls.name }}</td>
              <td>{{ cls.created_at[:10] if cls.created_at else '' }}</td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    {% endif %}
  </section>
{% endblock %}
```

### Шаг F8.2. Форма создания класса `templates/class_new.html`

Создайте файл `templates/class_new.html`:

```html
{% extends "base.html" %}

{% block title %}Новый класс · Мини‑ЛМС{% endblock %}

{% block content %}
  <section class="card">
    <h1>Новый класс</h1>

    {% if error %}
      <p class="bad">{{ error }}</p>
    {% endif %}

    <form method="post" action="{{ url_for('class_create') }}" class="form">
      <div class="form-group">
        <label for="name">Название класса</label>
        <input type="text" id="name" name="name" required />
      </div>

      <button type="submit" class="btn btn-primary">Создать класс</button>
    </form>

    <p style="margin-top: 16px;">
      <a href="{{ url_for('class_list') }}">К списку классов</a>
    </p>
  </section>
{% endblock %}
```

### Шаг F9. Администратор: создать пользователя с ролью

Администратор (master) должен иметь возможность создавать любых пользователей и **выбирать роль** (student, teacher, manager, admin). Таким образом, представление должно считывать значение `role` из формы и передавать его в функцию `create_user`, с резервным вариантом — значением `student` и проверкой данных.

**В файле `views_admin.py`, в функции `admin_user_create_view()`, замените блок, создающий пользователя.**:

```python
    username = ...
    password = ...
    role = (request.form.get("role") or "student").strip().lower()

    if not username or not password:
        ...

    if len(username) < 2:
        ...

    allowed_roles = ("student", "teacher", "manager", "admin")
    if role not in allowed_roles:
        role = "student"

    try:
        create_user(username, password, role)
        ...
```

### Шаг F10. Администратор: выпадающий список ролей в пользовательском интерфейсе — `templates/admin_users.html`

В шаблоне администратора добавьте выпадающий список **ролей** в форму создания пользователя, чтобы администратор мог выбрать студента, преподавателя, менеджера или администратора.

**В `templates/admin_users.html`, внутри формы, которая отправляет данные в `admin_user_create`, добавьте выпадающий список для выбора роли.** После поля пароля (и перед кнопкой отправки) добавьте:

```html
        <div class="form-group">
            <label for="new_role">Роль</label>
            <select id="new_role" name="role">
                <option value="student">Ученик (student)</option>
                <option value="teacher">Учитель (teacher)</option>
                <option value="manager">Менеджер (manager)</option>
                <option value="admin">Мастер (admin)</option>
            </select>
        </div>
```

---

## Итог Части F (ЛМС)

После выполнения этой части в проекте есть:

- **`db.py`**: таблица `classes` в `init_db()`.
- **`auth_utils.py`**: функции `is_manager()` и `is_master()`; в шаблонах они доступны через context_processor в `app.py`.
- **`views_auth.py`**: обновлённый `dashboard_view()` с подсчётом пользователей по ролям и передачей данных в ролевой шаблон дашборда.
- **`views_classes.py`**: хелпер `can_manage_classes()`, функции `class_list_view()`, `class_new_view()`, `class_create_view()`.
- **`app.py`**: маршруты `/classes`, `/classes/new` (GET и POST), делегирующие в `views_classes`.
- **Шаблоны**: ролевой `dashboard.html`, обновлённая навигация в `base.html`, `class_list.html`, `class_new.html`.

Это первый шаг предметной логики ЛМС: ролевой дашборд, хранение учебных групп в таблице `classes`, список и создание классов для менеджера и мастера. В следующих частях можно добавить привязку учеников к классам, занятия, домашние задания и статистику.
{% endraw %}
