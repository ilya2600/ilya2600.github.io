{% raw %}
# Часть E: Каналы и дашборд Slack‑чата

В Частях A–D вы:

- настроили базовый Flask‑сервер, шаблоны и стили;
- создали таблицу `users` и универсальный набор функций (`get_conn`, `init_db`, `create_user`, `ensure_master`, `current_user`, `is_logged_in`, `is_master`);
- добавили регистрацию и вход/выход, дашборд‑заглушку;
- вынесли общую шапку и навигацию в `base.html`;
- сделали админ‑страницу пользователей `/admin/users` с архивированием и удалением.

До этого момента проект был **универсальным** и не зависел от темы. В этой части мы начинаем предметную логику именно Slack‑чата:

- вводим первую сущность — **канал** (`channel`);
- делаем дашборд, который показывает список каналов;
- добавляем создание канала и страницу просмотра одного канала.

Сообщения, треды и управление участниками мы добавим в следующих частях.

---

## E1. Таблица `channels` в `init_db`

Сначала убедимся, что в базе есть таблица для каналов.

### 1. Смысл полей таблицы `channels`

Таблица `channels` будет хранить:

- `id` — идентификатор канала;
- `name` — имя канала (должно быть уникальным, чтобы легко ссылаться по имени);
- `type` — тип канала:
  - `'public'` — публичный, к нему может присоединиться любой пользователь;
  - `'private'` — приватный, только по приглашению;
  - `'read_only'` — канал только для чтения (писать могут только особые пользователи, позже разберём);
- `owner_id` — владелец канала (ссылка на `users.id`);
- `created_at` — дата/время создания канала.

### 2. Добавляем `CREATE TABLE channels` в `init_db`

Откройте `app.py` и найдите функцию `init_db()`. Внутри неё, после создания таблиц `settings` и `users`, добавьте блок:

```python
def init_db():
    """Создать все таблицы, если их ещё нет."""
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
            role TEXT NOT NULL CHECK (role IN ('user', 'admin')),
            archived_at TEXT,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        )
    """)

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

    conn.commit()
    conn.close()
```

На этом шаге мы **ещё не создаём** таблицы участников и сообщений — только каналы.

---

## E2. Дашборд `/dashboard` со списком каналов

Теперь сделаем так, чтобы после входа пользователь видел:

- каналы, где он уже состоит;
- публичные каналы, к которым он может присоединиться;
- кнопку «Создать канал».

Для этого нам понадобятся:

- дашборд‑маршрут `/dashboard`;
- вспомогательные функции авторизации, которые вы уже написали (`is_logged_in`, `current_user`, `is_master`).

### 1. Маршрут `GET /dashboard`

В `app.py` найдите текущий маршрут `dashboard` (он мог быть заглушкой из универсальной части). Замените его на такой вариант:

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

    # Каналы, где пользователь уже состоит
    my_channels = conn.execute("""
        SELECT c.id, c.name, c.type, c.owner_id
        FROM channels c
        JOIN channel_members cm ON c.id = cm.channel_id
        WHERE cm.user_id = ?
        ORDER BY c.name
    """, (user["id"],)).fetchall()

    # Публичные каналы, к которым пользователь может присоединиться
    public_joinable = conn.execute("""
        SELECT c.id, c.name, c.type
        FROM channels c
        WHERE c.type = 'public'
          AND c.id NOT IN (
            SELECT channel_id FROM channel_members WHERE user_id = ?
          )
        ORDER BY c.name
    """, (user["id"],)).fetchall()

    conn.close()

    return render_template(
        "dashboard.html",
        user=user,
        my_channels=my_channels,
        public_joinable=public_joinable,
    )
```

> Таблица `channel_members` мы будем вводить в следующей части. Если вы хотите в этой части не использовать её вообще, можно временно считать, что владелец автоматически «участник», и выбирать только по `channels.owner_id = user["id"]`. В этой инструкции мы сразу ориентируемся на связку каналов и участников.

### 2. Шаблон `dashboard.html` для списка каналов

Создайте или перепишите `templates/dashboard.html`, чтобы он показывал два списка:

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
      <p class="muted">Вы ещё не состоите ни в одном канале.</p>
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
      <p class="muted">Нет доступных публичных каналов, к которым вы ещё не присоединились.</p>
    {% else %}
      <ul>
        {% for ch in public_joinable %}
          <li>
            <a href="{{ url_for('channel_join', channel_id=ch.id) }}">Вступить в #{{ ch.name }}</a>
          </li>
        {% endfor %}
      </ul>
    {% endif %}
  </section>
{% endblock %}
```

На этом шаге кнопки «Создать канал» и «Вступить» ещё не работают — мы подключим соответствующие маршруты дальше.

---

## E3. Создание канала `/channels/new`

Теперь добавим форму и маршруты для создания нового канала.

### 1. Маршруты `GET /channels/new` и `POST /channels/new`

В `app.py` добавьте:

```python
@app.get("/channels/new")
def channel_new():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    return render_template("channel_new.html", error=None)


@app.post("/channels/new")
def channel_create():
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
        # В следующей части можно будет сразу добавить владельца в channel_members
        flash("Канал создан.")
    except sqlite3.IntegrityError:
        flash("Канал с таким именем уже существует.")
    conn.close()

    return redirect(url_for("dashboard"))
```

Здесь:

- любой вошедший пользователь может создать канал;
- имя канала должно быть уникальным, иначе получаем `IntegrityError`;
- тип канала выбирается из ограниченного списка.

### 2. Шаблон `channel_new.html`

Создайте файл `templates/channel_new.html`:

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

После этого кнопка «Создать канал» на дашборде начнёт работать.

---

## E4. Просмотр одного канала `/channels/<id>`

Сделаем страницу, на которой можно увидеть информацию о канале. Сообщения и треды добавим позже.

### 1. Маршрут `GET /channels/<int:channel_id>`

В `app.py` добавьте:

```python
@app.get("/channels/<int:channel_id>")
def channel_view(channel_id):
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

На этом шаге мы не проверяем членство в канале — это сделаем в следующих частях, когда введём таблицу `channel_members` и хелперы прав. Пока допустим, что все каналы условно «видимы» (или ограничим типами при необходимости).

### 2. Шаблон `channel_view.html`

Создайте файл `templates/channel_view.html`:

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

Так пользователь уже видит отдельную страницу канала с базовой информацией.

---

## E5. Навигация в `base.html` (минимальные доработки)

В универсальной части вы уже сделали `base.html` с:

- ссылками на главную, дашборд, вход/выход, регистрацию;
- ссылками на админ‑разделы для мастера.

Для Slack‑чата можно оставить навигацию почти без изменений, добавив только понятную надпись «Slack‑чат» в заголовке и убедившись, что ссылка на дашборд ведёт к списку каналов.

Пример упрощённого заголовка в `base.html`:

```html
<header>
  <h1><a href="{{ url_for('home') }}">Slack‑чат (учебный)</a></h1>
  <nav>
    <a href="{{ url_for('home') }}">Главная</a>

    {% if current_user() %}
      <a href="{{ url_for('dashboard') }}">Каналы</a>
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
  <hr />
</header>
```

Этого достаточно, чтобы пользователь понимал: главная «рабочая» точка приложения — список каналов на дашборде.

---

## Итог Части E

После выполнения этой части у вас есть:

- таблица `channels` с полями имени, типа, владельца и датой создания;
- дашборд `/dashboard`, который показывает:
  - каналы, где пользователь уже состоит,
  - доступные публичные каналы для вступления,
  - кнопку «Создать канал»;
- форма создания канала `/channels/new`;
- страница просмотра одного канала `/channels/<id>` с базовой информацией;
- навигация в `base.html`, которая подчёркивает, что каналы — центральная часть Slack‑чата.

В следующих частях вы добавите:

- таблицу участников каналов (`channel_members`) и права доступа;
- отправку и отображение сообщений (`messages`), треды;
- типы каналов (`public`/`private`/`read_only`) и soft delete сообщений.
{% endraw %}

