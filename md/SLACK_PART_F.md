{% raw %}
# Часть F: Каналы — первая предметная сущность

В Частях A–E вы разнесли приложение по модулям:

- `app.py` — создание Flask‑приложения и тонкие маршруты;
- `db.py` — работа с базой (`init_db`, `get_conn` и т.п.);
- `auth_utils.py` — пользователи и права (`current_user`, `is_admin`, `is_logged_in`, `get_registration_open`);
- `views_auth.py` — логика входа, регистрации и дашборда;
- `views_admin.py` — логика админ‑страниц;
- шаблоны в `templates/`.

В этой части вы добавляете **предметную логику Slack‑чата**:

- таблицу `channels` в `db.py`;
- модуль `views_channels.py` с логикой дашборда (список каналов), создания канала и просмотра одного канала;
- маршруты `/dashboard` (Slack‑версия), `/channels/new`, `/channels/<id>` в `app.py`;
- шаблоны дашборда со списком каналов, формы создания канала и страницы просмотра канала;
- обновлённую навигацию в `base.html` в соответствии с текущим контекстом проекта (`registration_open`, `is_admin()`).

Таблицу участников каналов (`channel_members`), вступление в каналы и сообщения оставляем на следующие части.

---

## F1. Таблица `channels` в `db.py`

Первая предметная сущность Slack‑чата — канал.

### Шаг F1.1. Поля таблицы `channels`

Канал хранит: идентификатор, уникальное имя, тип (`public` / `private` / `read_only`), владельца (ссылка на `users.id`), дату создания.

### Шаг F1.2. Добавляем таблицу в `init_db()`

Откройте `db.py` и найдите функцию `init_db()`. Сразу **после** блока `CREATE TABLE IF NOT EXISTS users (...)` добавьте создание таблицы `channels`:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS channels (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL UNIQUE,
            type TEXT NOT NULL CHECK (type IN ('public', 'private', 'read_only')),
            owner_id INTEGER,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            FOREIGN KEY (owner_id) REFERENCES users(id)
        )
    """)
```

В итоге `init_db()` создаёт таблицы `settings`, `users`, `channels`, затем выполняет `conn.commit()` и `conn.close()`.

Таблицу `channel_members` и сообщения в этой части не добавляем.

---

## F2. Модуль `views_channels.py` — дашборд и каналы

Вся логика дашборда со списком каналов, создания канала и просмотра одного канала выносится в отдельный модуль.

### Шаг F2.1. Создаём файл `views_channels.py`

В той же папке, где лежат `app.py`, `views_auth.py`, `views_admin.py`, создайте файл `views_channels.py`.

### Шаг F2.2. Импорты

В начало `views_channels.py` вставьте:

```python
import sqlite3
from flask import render_template, redirect, url_for, request, abort, flash, session

from db import get_conn
from auth_utils import is_logged_in, current_user
```

### Шаг F2.3. Дашборд со списком каналов

В этой части под «моими каналами» считаем только каналы, где текущий пользователь — владелец (`owner_id`). Таблицу `channel_members` и вступление в каналы добавим позже. Публичные каналы, к которым можно «присоединиться», — это каналы с типом `public`, которыми пользователь не владеет.

Добавьте в `views_channels.py`:

```python
def dashboard_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    if user is None:
        session.clear()
        return redirect(url_for("login_form"))

    conn = get_conn()

    # Каналы, где пользователь — владелец (пока без channel_members)
    my_channels = conn.execute("""
        SELECT id, name, type, owner_id
        FROM channels
        WHERE owner_id = ?
        ORDER BY name
    """, (user["id"],)).fetchall()

    # Публичные каналы, которыми пользователь не владеет (доступны для просмотра/вступления позже)
    public_joinable = conn.execute("""
        SELECT id, name, type
        FROM channels
        WHERE type = 'public' AND (owner_id IS NULL OR owner_id != ?)
        ORDER BY name
    """, (user["id"],)).fetchall()

    conn.close()

    return render_template(
        "dashboard.html",
        user=user,
        my_channels=my_channels,
        public_joinable=public_joinable,
    )
```

### Шаг F2.4. Создание канала

Добавьте две функции для формы и обработки создания канала:

```python
def channel_new_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    return render_template("channel_new.html", error=None)


def channel_create_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    name = (request.form.get("name") or "").strip()
    ctype = request.form.get("type") or "public"

    if not name:
        return render_template("channel_new.html", error="Введите имя канала.")

    if ctype not in ("public", "private", "read_only"):
        ctype = "public"

    user = current_user()
    conn = get_conn()
    try:
        conn.execute(
            "INSERT INTO channels (name, type, owner_id) VALUES (?, ?, ?)",
            (name, ctype, user["id"]),
        )
        conn.commit()
        flash("Канал создан.")
    except sqlite3.IntegrityError:
        flash("Канал с таким именем уже существует.")
    conn.close()

    return redirect(url_for("dashboard"))
```

Имя канала должно быть уникальным; при дубликате обрабатываем `IntegrityError` и перенаправляем на дашборд с сообщением.

### Шаг F2.5. Просмотр одного канала

Добавьте функцию просмотра страницы канала (проверку членства в канале добавим в следующих частях):

```python
def channel_view_view(channel_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    conn = get_conn()
    ch = conn.execute("""
        SELECT
            c.id,
            c.name,
            c.type,
            c.owner_id,
            c.created_at,
            u.username AS owner_name
        FROM channels c
        LEFT JOIN users u ON c.owner_id = u.id
        WHERE c.id = ?
    """, (channel_id,)).fetchone()
    conn.close()

    if ch is None:
        abort(404)

    return render_template("channel_view.html", channel=ch)
```

---

## F3. Подключение дашборда и маршрутов каналов в `app.py`

Для Slack‑проекта дашборд должен показывать каналы, поэтому маршрут `/dashboard` вызывает реализацию из `views_channels`, а не из `views_auth`.

### Шаг F3.1. Импорты

В `app.py`:

1. Добавьте импорт из `views_channels` и замените импорт `dashboard_view`: пусть `dashboard_view` приходит из `views_channels`, а не из `views_auth`.

Например, было:

```python
from views_auth import (
    login_form_view,
    login_view,
    logout_view,
    register_form_view,
    register_view,
    dashboard_view,
)
```

Сделайте:

```python
from views_auth import (
    login_form_view,
    login_view,
    logout_view,
    register_form_view,
    register_view,
)
from views_channels import (
    dashboard_view,
    channel_new_view,
    channel_create_view,
    channel_view_view,
)
```

Маршрут `@app.get("/dashboard")` по‑прежнему вызывает `dashboard_view()` — теперь это реализация из `views_channels.py` со списком каналов.

**Примечание.** После этого функция `dashboard_view` в `views_auth.py` для Slack‑варианта не используется. Её можно оставить для других вариантов проекта или не импортировать из `views_auth` при сборке Slack‑чата.

2. Добавьте маршруты каналов (ниже маршрутов админки или после `/dashboard`):

```python
@app.get("/channels/new")
def channel_new():
    return channel_new_view()


@app.post("/channels/new")
def channel_create():
    return channel_create_view()


@app.get("/channels/<int:channel_id>")
def channel_view(channel_id):
    return channel_view_view(channel_id)
```

Вся проверка прав и работа с БД остаётся в `views_channels.py`.

---

## F4. Навигация: соответствие текущему контексту проекта

В текущем проекте в `app.py` context_processor уже передаёт в шаблоны:

- `current_user` — функция;
- `is_admin` — функция;
- `registration_open` — **значение** (результат вызова `get_registration_open()`), а не сама функция.

Поэтому в шаблонах навигации нужно использовать **переменную** `registration_open`, а не вызов `get_registration_open()` (иначе Jinja выдаст ошибку — функции с таким именем в контексте нет). Для ссылки на страницу регистрации используйте имя маршрута, объявленного в `app.py`: для GET `/register` это `register_form`, т.е. `url_for('register_form')`.

Для админ‑ссылок (Настройки, Пользователи) в проекте уже доступна функция `is_admin()` — используйте её в навигации. Добавлять отдельно `is_master()` не обязательно; при желании можно завести в `auth_utils.py` алиас `is_master = is_admin` и передать его в context_processor, но для соответствия текущему контексту достаточно `is_admin()` в шаблонах.

---

## F5. Шаблон дашборда `dashboard.html`

Дашборд показывает два списка: «мои каналы» и «доступные публичные каналы», а также кнопку «Создать канал».

### Шаг F5.1. Создаём или заменяем `templates/dashboard.html`

Содержимое файла `templates/dashboard.html`:

```html
{% extends "base.html" %}

{% block title %}Дашборд · Slack‑чат{% endblock %}

{% block content %}
  <section class="card">
    <h1>Дашборд</h1>
    <p>Вы вошли как <strong>{{ user.username }}</strong> (роль: {{ user.role }}).</p>

    <p>
      <a href="{{ url_for('channel_new') }}" class="btn btn-primary">Создать канал</a>
    </p>

    <h2>Мои каналы</h2>
    {% if not my_channels %}
      <p class="muted">Вы ещё не создали ни одного канала.</p>
    {% else %}
      <ul>
        {% for ch in my_channels %}
          <li>
            <a href="{{ url_for('channel_view', channel_id=ch.id) }}">#{{ ch.name }}</a>
            {% if ch.type == 'read_only' %}<span class="badge">read‑only</span>{% endif %}
          </li>
        {% endfor %}
      </ul>
    {% endif %}

    <h2>Доступные публичные каналы</h2>
    {% if not public_joinable %}
      <p class="muted">Нет доступных публичных каналов.</p>
    {% else %}
      <ul>
        {% for ch in public_joinable %}
          <li>
            <a href="{{ url_for('channel_view', channel_id=ch.id) }}">#{{ ch.name }}</a>
            (просмотр; вступление — в следующих частях)
          </li>
        {% endfor %}
      </ul>
    {% endif %}
  </section>
{% endblock %}
```

Переменные `user`, `my_channels`, `public_joinable` передаёт `dashboard_view()` из `views_channels.py`. Ссылка «Вступить» из Part E здесь заменена на переход к просмотру канала; вступление и таблица `channel_members` будут в следующих частях.

---

## F6. Шаблон создания канала `channel_new.html`

### Шаг F6.1. Создаём файл `templates/channel_new.html`

```html
{% extends "base.html" %}

{% block title %}Новый канал · Slack‑чат{% endblock %}

{% block content %}
  <section class="card">
    <h1>Новый канал</h1>

    {% if error %}
      <p class="bad">{{ error }}</p>
    {% endif %}

    <form method="post" action="{{ url_for('channel_create') }}" class="form">
      <div class="form-group">
        <label for="name">Имя канала</label>
        <input
          type="text"
          id="name"
          name="name"
          required
          placeholder="например, general"
        />
      </div>

      <div class="form-group">
        <label for="type">Тип канала</label>
        <select id="type" name="type">
          <option value="public">Публичный</option>
          <option value="private">Приватный</option>
          <option value="read_only">Только чтение</option>
        </select>
      </div>

      <button type="submit" class="btn btn-primary">Создать канал</button>
    </form>

    <p style="margin-top: 16px;">
      <a href="{{ url_for('dashboard') }}">К дашборду</a>
    </p>
  </section>
{% endblock %}
```

---

## F7. Шаблон просмотра канала `channel_view.html`

### Шаг F7.1. Создаём файл `templates/channel_view.html`

```html
{% extends "base.html" %}

{% block title %}Канал #{{ channel.name }} · Slack‑чат{% endblock %}

{% block content %}
  <section class="card">
    <h1>#{{ channel.name }}</h1>

    <p class="muted">
      Тип: {{ channel.type }}<br />
      Владелец:
      {% if channel.owner_name %}
        {{ channel.owner_name }}
      {% else %}
        (не указан)
      {% endif %}
      <br />
      Создан: {{ channel.created_at }}
    </p>

    <h2>Сообщения</h2>
    <p class="muted">
      Здесь позже появятся сообщения и треды этого канала.
    </p>

    <p style="margin-top: 16px;">
      <a href="{{ url_for('dashboard') }}">К дашборду</a>
    </p>
  </section>
{% endblock %}
```

---

## F8. Навигация в `base.html` для Slack‑чата

Навигация должна использовать только то, что уже передаётся в шаблоны через context_processor в `app.py`: переменную `registration_open` и функцию `is_admin()`. Имена маршрутов должны совпадать с объявлениями в `app.py` (например, для страницы регистрации — `register_form`).

Обновите блок навигации в `templates/base.html`, например:

```html
<header>
  <h1><a href="{{ url_for('home') }}">Slack‑чат (учебный)</a></h1>
  <nav>
    <a href="{{ url_for('home') }}">Главная</a>

    {% if current_user() %}
      <a href="{{ url_for('dashboard') }}">Каналы</a>
      {% if is_admin() %}
        <a href="{{ url_for('admin_settings') }}">Настройки</a>
        <a href="{{ url_for('admin_users') }}">Пользователи</a>
      {% endif %}
      <span style="float: right;">
        {{ current_user().username }}
        (<a href="{{ url_for('logout') }}">выйти</a>)
      </span>
    {% else %}
      <a href="{{ url_for('login_form') }}">Вход</a>
      {% if registration_open %}
        <a href="{{ url_for('register_form') }}">Регистрация</a>
      {% endif %}
    {% endif %}
  </nav>
  <hr />
</header>
```

- Для отображения ссылки «Регистрация» используется переменная **`registration_open`** (она передаётся в контекст в `app.py` как результат `get_registration_open()`). Вызов `get_registration_open()` в шаблоне использовать нельзя — такой функции в контексте нет.
- Ссылка на страницу регистрации ведёт на маршрут с именем **`register_form`** (GET `/register` в `app.py`).
- Админ‑блок показывается по **`is_admin()`**, как в текущем проекте.

---

## Итог Части F (Slack)

После выполнения этой части в проекте есть:

- **`db.py`**: таблица `channels` в `init_db()` с полями `id`, `name` (UNIQUE), `type`, `owner_id`, `created_at`.
- **`views_channels.py`**: функции `dashboard_view()` (список «моих» каналов по владельцу и публичных каналов), `channel_new_view()`, `channel_create_view()`, `channel_view_view()`.
- **`app.py`**: импорт `dashboard_view` и маршрутов каналов из `views_channels`; маршруты `/dashboard`, `/channels/new` (GET и POST), `/channels/<id>`. Функция `dashboard_view` в `views_auth.py` при этом для Slack‑варианта не используется.
- **Шаблоны**: `dashboard.html` (список каналов), `channel_new.html`, `channel_view.html`; навигация в `base.html` использует `registration_open`, `is_admin()` и имена маршрутов (`register_form`, `login_form` и т.д.) в соответствии с текущим контекстом проекта.

Это первый шаг предметной логики Slack‑чата: каналы, дашборд со списком каналов, создание канала и просмотр одного канала. В следующих частях можно добавить таблицу `channel_members`, вступление в каналы, сообщения и треды.
{% endraw %}
