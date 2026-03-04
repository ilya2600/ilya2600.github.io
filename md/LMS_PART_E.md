{% raw %}
# Часть E: Ролевой дашборд и первые «классы» в ЛМС

В Частях A–D вы:

- построили базовое Flask‑приложение с шаблонами и стилями;
- создали таблицу `users`, регистрацию, вход/выход, мастер‑аккаунт;
- добавили флаг «регистрация открыта/закрыта» и страницу `/admin/settings`;
- сделали админ‑страницу `/admin/users` с архивированием и удалением пользователей;
- перевели основные страницы на базовый шаблон `base.html`.

Во всех проектах эти части были **универсальными** — нигде ещё не говорилось о «классах», «уроках» и т.п.

В этой части мы начинаем предметную логику ЛМС:

- делаем **ролевой дашборд**, который показывает разный текст и навигацию для учеников, учителей, менеджеров и мастера;
- вводим первую простую сущность ЛМС — **класс** (учебная группа): список классов и форма создания классов для менеджера.

Домашние задания, занятия, посещаемость и статистика появятся в следующих частях.

---

## E1. Ролевой дашборд `/dashboard`

Сейчас дашборд из универсальных частей — просто заглушка «Здравствуйте, &lt;имя&gt;». Для ЛМС сделаем:

- разные тексты в зависимости от роли;
- небольшой обзор того, для чего эта роль нужна в системе;
- простую статистику по ролям (по таблице `users`).

### 1. Обновляем маршрут `/dashboard`

Откройте `app.py` и найдите маршрут `dashboard`. Замените его реализацию на такую:

```python
@app.get("/dashboard")
def dashboard():
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

Здесь мы:

- гарантируем, что пользователь авторизован;
- считаем, сколько активных (`archived_at IS NULL`) пользователей каждой роли есть в системе;
- передаём эти цифры в шаблон, чтобы показать простую «панель» для менеджера/мастера.

### 2. Обновляем шаблон `dashboard.html`

Перепишем `templates/dashboard.html`, чтобы он:

- наследовал базовый шаблон,
- показывал разные блоки в зависимости от роли.

Создайте или замените содержимое `templates/dashboard.html`:

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
        <!-- Позже: ссылки на занятия, статистику и т.п. -->
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

Так студент видит, что роли в ЛМС «живые» и от них зависит, что показывает система.

---

## E2. Навигация `base.html` с учётом ЛМС‑ролей

Сейчас `base.html` показывает одинаковое меню для всех вошедших пользователей. Для ЛМС сделаем:

- базовый набор ссылок для всех;
- дополнительные ссылки для менеджера и мастера.

### 1. Обновляем навигацию в `base.html`

Откройте `templates/base.html` и найдите блок `<nav>...</nav>`. Замените его внутри на вариант с учётом ролей:

```html
<nav>
  <a href="{{ url_for('home') }}">Главная</a>

  {% if current_user() %}
    <a href="{{ url_for('dashboard') }}">Дашборд</a>

    {# Для менеджера и мастера: ссылка на классы #}
    {% if is_manager() or is_master() %}
      <a href="{{ url_for('class_list') }}">Классы</a>
    {% endif %}

    {# Админские ссылки оставляем только мастеру #}
    {% if is_master() %}
      <a href="{{ url_for('admin_settings') }}">Настройки</a>
      <a href="{{ url_for('admin_users') }}">Пользователи</a>
    {% endif %}

    <span style="float: right;">
      {{ current_user().username }}
      (<a href="{{ url_for('logout') }}">выйти</a>)
    </span>
  {% else %}
    <a href="{{ url_for('login_form') }}">Вход</a>
    {% if get_registration_open() %}
      <a href="{{ url_for('register') }}">Регистрация</a>
    {% endif %}
  {% endif %}
</nav>
```

> Здесь используются функции `is_manager()` и `is_master()`, которые вы добавили ранее в универсальной части.

Так мы готовим почву для следующего шага: отдельной страницы «Классы» только для тех ролей, которые ими управляют.

---

## E3. Таблица `classes` в `init_db` (пояснение)

В финальной версии ЛМС таблица `classes` хранит учебные группы:

- `id` — идентификатор класса;
- `name` — название группы (например, «7А», «Группа Python 1»);
- `created_at` — дата/время создания.

Если вы ещё не добавляли эту таблицу, расширьте `init_db()` следующим блоком (после `users` и до других предметных таблиц):

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
            role TEXT NOT NULL CHECK (role IN ('student', 'teacher', 'manager', 'admin')),
            archived_at TEXT,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        )
    """)

    conn.execute("""
        CREATE TABLE IF NOT EXISTS classes (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        )
    """)

    conn.commit()
    conn.close()
```

В этой части мы **не используем** ещё таблицы `class_students`, `lessons`, `homework` и т.п. — только `classes` как первую доменную сущность.

---

## E4. Список классов `/classes` (для менеджера и мастера)

Сделаем страницу, где менеджер/мастер видит все классы.

### 1. Маршрут `GET /classes`

Добавьте в `app.py`:

```python
@app.get("/classes")
def class_list():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    # Управлять классами могут менеджер и мастер
    if not is_manager() and not is_master():
        abort(403)

    conn = get_conn()
    classes = conn.execute(
        "SELECT id, name, created_at FROM classes ORDER BY name"
    ).fetchall()
    conn.close()

    return render_template("class_list.html", classes=classes)
```

### 2. Шаблон `class_list.html`

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

Этого достаточно, чтобы менеджер видел список всех учебных групп.

---

## E5. Создание нового класса `/classes/new`

Теперь добавим форму для создания класса.

### 1. Маршруты `GET /classes/new` и `POST /classes/new`

В `app.py` добавьте:

```python
@app.get("/classes/new")
def class_new():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    if not is_manager() and not is_master():
        abort(403)

    return render_template("class_new.html", error=None)


@app.post("/classes/new")
def class_create():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    if not is_manager() and not is_master():
        abort(403)

    name = (request.form.get("name") or "").strip()

    if not name:
        flash("Введите название класса.")
        return render_template("class_new.html", error="Введите название класса.")

    conn = get_conn()
    conn.execute(
        "INSERT INTO classes (name) VALUES (?)",
        (name,),
    )
    conn.commit()
    conn.close()

    flash("Класс создан.")
    return redirect(url_for("class_list"))
```

### 2. Шаблон `class_new.html`

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

Теперь менеджер (и мастер) могут заходить в `/classes`, видеть список классов и создавать новые.

---

## Итог Части E

После выполнения этой части у вас есть:

- **ролевой дашборд** `/dashboard`, который показывает разные блоки для учеников, учителей, менеджеров и мастера, а также простую статистику по ролям;
- обновлённая навигация в `base.html`, где для менеджера и мастера появляется ссылка «Классы»;
- таблица `classes` в базе данных;
- список классов `/classes` и форма создания нового класса `/classes/new`, доступные только менеджеру и мастеру.

Это первый шаг предметной логики ЛМС: система умеет хранить и показывать учебные группы. В следующих частях вы добавите:

- привязку учеников к классам;
- планирование занятий (`lessons`) для классов;
- домашние задания, отправку работ, посещаемость и статистику.
{% endraw %}

