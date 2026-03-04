{% raw %}
# Часть C: Регистрация и общий макет

В этой части вы:

- добавите настройку «регистрация открыта/закрыта», которой управляет только мастер (администратор),
- сделаете страницу регистрации новых пользователей,
- вынесете общую шапку и навигацию в базовый шаблон.

После Части C гость сможет зарегистрироваться (если мастер включил регистрацию), а все основные страницы будут наследовать единый макет. Код HTML станет аккуратнее: общие части вынесены в один файл.

**Предполагается, что вы уже сделали Части A и B:**

- есть Flask‑приложение с маршрутами `/`, `/login`, `/logout`, `/dashboard`;
- таблица `users` и функции работы с БД (`get_conn`, `init_db`, `create_user`, `ensure_master`, `current_user`, `is_logged_in`, `is_master`);
- включена сессия (`app.secret_key`), реализованы вход и выход из системы;
- есть простая защищённая страница‑дашборд (`/dashboard`).

---

## C1. Настройка «регистрация открыта/закрыта» для мастера

### 1. Хранение настроек в таблице `settings`

Уже есть таблица `settings` с парой ключ–значение (`key`, `value`). В ней удобно хранить любые настройки.

Для регистрации используем ключ вроде `"registration_open"` с возможными значениями:

- `"1"` — регистрация разрешена,
- `"0"` — регистрация закрыта.

В `init_db()` таблица `settings` уже создаётся и в неё вставляется запись `'registration_open' = '0'` (по умолчанию регистрация закрыта).

### 2. Хелпер для чтения настройки регистрации

Добавим функцию, которая будет возвращать булево значение:

```python
def get_registration_open():
    conn = get_conn()
    row = conn.execute("SELECT value FROM settings WHERE key = 'registration_open'").fetchone()
    conn.close()
    return row is not None and row["value"] == "1"
```

Логика:

- если записи нет или значение не `"1"` — считаем, что регистрация закрыта;
- если есть строка со значением `"1"` — регистрация открыта.

Эта функция уже используется в нескольких местах (формы регистрации, настройки) и доступна в шаблонах.

### 3. Страница настроек только для мастера

Сделаем маршруты, которые позволяют мастеру включать/выключать регистрацию:

- `GET /admin/settings` — показывает текущий статус и форму;
- `POST /admin/settings` — сохраняет новое значение.

В `app.py` уже есть вспомогательные функции `is_logged_in()` и `is_master()`. Маршруты выглядят так:

```python
@app.get("/admin/settings")
def admin_settings():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_master():
        abort(403)
    return render_template("admin_settings.html", registration_open=get_registration_open())


@app.post("/admin/settings")
def admin_settings_save():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_master():
        abort(403)
    open_val = "1" if request.form.get("registration_open") == "on" else "0"
    conn = get_conn()
    conn.execute("REPLACE INTO settings (key, value) VALUES ('registration_open', ?)", (open_val,))
    conn.commit()
    conn.close()
    return redirect(url_for("admin_settings"))
```

Обратите внимание:

- неавторизованных всегда перенаправляем на форму входа;
- если пользователь вошёл, но не является мастером — возвращаем код 403 (`abort(403)`).

### 4. Шаблон `admin_settings.html`

Создайте файл `templates/admin_settings.html`, который показывает простую форму с чекбоксом:

```html
<!DOCTYPE html>
<html lang="ru">
  <head>
    <meta charset="utf-8" />
    <title>Настройки</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='styles.css') }}" />
  </head>
  <body>
    <div class="card">
      <h1>Настройки</h1>

      <form method="post" action="{{ url_for('admin_settings_save') }}">
        <label>
          <input
            type="checkbox"
            name="registration_open"
            {% if registration_open %}checked{% endif %}
          />
          Разрешить регистрацию новых пользователей
        </label>

        <div style="margin-top: 12px;">
          <button type="submit">Сохранить</button>
        </div>
      </form>

      <p style="margin-top: 16px;">
        <a href="{{ url_for('dashboard') }}">Вернуться в дашборд</a>
      </p>
    </div>
  </body>
</html>
```

**Проверка C1:**

1. Войдите под мастером.
2. Откройте `/admin/settings`.
3. Включите и выключите чекбокс, обновите страницу несколько раз — состояние должно сохраняться.

---

## C2. Регистрация новых пользователей

Теперь добавим регистрацию для новых пользователей.

Правила:

- регистрация доступна только для неавторизованных пользователей;
- страница регистрации вообще работает только если мастер включил флаг `registration_open`;
- при закрытой регистрации любая попытка открыть `/register` или отправить форму даёт понятную ошибку;
- при открытой регистрации создаётся пользователь с базовой ролью (например, `agent` в текущей схеме), пароль хешируется.

### 1. Маршруты регистрации

В `app.py` уже есть два маршрута `/register`:

```python
@app.get("/register")
def register_form():
    if session.get("user_id") is not None:
        return redirect(url_for("dashboard"))
    if not get_registration_open():
        return render_template("register.html", registration_disabled=True), 403
    return render_template("register.html")


@app.post("/register")
def register():
    if not get_registration_open():
        return render_template("register.html", registration_disabled=True), 403
    username = (request.form.get("username") or "").strip()
    password = request.form.get("password") or ""
    if not username or not password:
        return render_template("register.html", error="Username and password required."), 400
    if len(username) < 2:
        return render_template("register.html", error="Username too short."), 400
    conn = get_conn()
    try:
        conn.execute(
            "INSERT INTO users (username, password_hash, role) VALUES (?, ?, 'agent')",
            (username, generate_password_hash(password)),
        )
        conn.commit()
    except sqlite3.IntegrityError:
        conn.close()
        return render_template("register.html", error="Username already taken."), 409
    conn.close()
    return redirect(url_for("login_form"))
```

Логика:

- гости могут открыть `/register` только если `get_registration_open()` вернул `True`;
- при POST обрабатываются:
  - пустой логин или пароль,
  - слишком короткий логин,
  - занятый логин (`sqlite3.IntegrityError` из‑за `UNIQUE(username)`),
  - нормальный случай — создание пользователя с ролью по умолчанию.

### 2. Шаблон `register.html`

Создайте файл `templates/register.html` с формой регистрации и обработкой ошибок:

```html
<!DOCTYPE html>
<html lang="ru">
  <head>
    <meta charset="utf-8" />
    <title>Регистрация</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='styles.css') }}" />
  </head>
  <body>
    <div class="card">
      <h1>Регистрация</h1>

      {% if registration_disabled %}
        <p>Регистрация временно закрыта. Попробуйте позже или обратитесь к администратору.</p>
        <p><a href="{{ url_for('home') }}">На главную</a></p>
      {% else %}
        {% if error %}
          <p style="color: darkred;">{{ error }}</p>
        {% endif %}

        <form method="post" action="{{ url_for('register') }}">
          <div>
            <label for="username">Логин</label><br />
            <input id="username" name="username" required />
          </div>

          <div style="margin-top: 8px;">
            <label for="password">Пароль</label><br />
            <input id="password" name="password" type="password" required />
          </div>

          <div style="margin-top: 12px;">
            <button type="submit">Создать аккаунт</button>
          </div>
        </form>

        <p style="margin-top: 12px;">
          Уже есть аккаунт?
          <a href="{{ url_for('login_form') }}">Войти</a>
        </p>
      {% endif %}
    </div>
  </body>
</html>
```

### 3. Ссылки на регистрацию

Чтобы регистрация не была «спрятана», добавьте ссылки:

- на главной странице (`home.html`) рядом с предложением войти — «или зарегистрируйтесь» (если флаг регистрации включён);
- на странице входа (`login.html`) — «Нет аккаунта? Зарегистрируйтесь».

Пример для `login.html`:

```html
<p style="margin-top: 12px;">
  Нет аккаунта?
  <a href="{{ url_for('register') }}">Зарегистрируйтесь</a>.
</p>
```

**Проверка C2:**

1. Убедитесь, что в `/admin/settings` включён флаг регистрации.
2. Выйдите из системы.
3. Откройте `/register`, зарегистрируйтесь под новым логином.
4. Убедитесь, что новый пользователь может войти.
5. Отключите регистрацию и попробуйте снова открыть `/register` — должна появиться ошибка о закрытой регистрации.

---

## C3. Общий макет для страниц (базовый шаблон)

Сейчас каждая HTML‑страница содержит повторяющиеся части:

- `<!DOCTYPE html>`, `<html>`, `<head>`, подключение CSS;
- заголовок приложения;
- навигацию: ссылки на главную, вход, выход, регистрацию, настройки.

Чтобы не дублировать разметку, вынесем общий каркас в шаблон `base.html`, а остальные файлы будут **наследовать** его.

### 1. Контекстный процессор для шаблонов

Чтобы в любом шаблоне было удобно использовать `current_user`, `is_master`, `is_agent` и текущий флаг регистрации, в `app.py` уже добавлен контекстный процессор:

```python
@app.context_processor
def inject():
    return {
        "current_user": current_user,
        "is_master": is_master,
        "is_agent": is_agent,
        "registration_open": get_registration_open(),
    }
```

Благодаря этому:

- в шаблонах можно вызывать `current_user()` прямо в `if`‑ах и выражениях;
- доступна булевая переменная `registration_open` для показа/скрытия ссылок.

### 2. Создаём `base.html`

Создайте файл `templates/base.html`:

```html
<!DOCTYPE html>
<html lang="ru">
  <head>
    <meta charset="utf-8" />
    <title>{% block title %}Учебное веб‑приложение{% endblock %}</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='styles.css') }}" />
  </head>
  <body>
    <header>
      <h1><a href="{{ url_for('home') }}">Учебное веб‑приложение</a></h1>
      <nav>
        <a href="{{ url_for('home') }}">Главная</a>

        {% if current_user() %}
          <a href="{{ url_for('dashboard') }}">Дашборд</a>

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
          {% if registration_open %}
            <a href="{{ url_for('register') }}">Регистрация</a>
          {% endif %}
        {% endif %}
      </nav>
      <hr />
    </header>

    <main>
      {% block content %}{% endblock %}
    </main>
  </body>
</html>
```

Здесь:

- блок `title` позволяет страницам задавать свой заголовок вкладки;
- в навигации показаны разные ссылки для гостя и для вошедшего пользователя;
- при наличии прав мастера добавлены ссылки на административные разделы.

### 3. Перевод существующих шаблонов на наследование

Теперь перепишем основные шаблоны так, чтобы они наследовали `base.html`.

#### `home.html`

Было (упрощённо): отдельный `<html>`, `<head>`, `<body>`. Замените содержимое полностью на:

```html
{% extends "base.html" %}

{% block title %}Главная · Учебное веб‑приложение{% endblock %}

{% block content %}
  <div class="card">
    <p>База данных:
      {% if db_ok %}
        <strong>OK</strong>
      {% else %}
        Ошибка
      {% endif %}
    </p>

    {% if current_user() %}
      <p>
        Добро пожаловать, {{ current_user().username }}.
        Перейдите в <a href="{{ url_for('dashboard') }}">дашборд</a>.
      </p>
    {% else %}
      <p>Вы гость. Вы можете
        <a href="{{ url_for('login_form') }}">войти</a>
        {% if registration_open %}
          или <a href="{{ url_for('register') }}">зарегистрироваться</a>.
        {% endif %}
      </p>
    {% endif %}
  </div>
{% endblock %}
```

#### `login.html`

Перепишите полностью шаблон входа так:

```html
{% extends "base.html" %}

{% block title %}Вход · Учебное веб‑приложение{% endblock %}

{% block content %}
  <div class="card">
    <h2>Вход</h2>

    {% if error %}
      <p style="color: darkred;">{{ error }}</p>
    {% endif %}

    <form method="post" action="{{ url_for('login') }}">
      <input type="hidden" name="next" value="{{ request.args.get('next', '') }}" />

      <div>
        <label for="username">Логин</label><br />
        <input id="username" name="username" required />
      </div>

      <div style="margin-top: 8px;">
        <label for="password">Пароль</label><br />
        <input id="password" name="password" type="password" required />
      </div>

      <div style="margin-top: 12px;">
        <button type="submit">Войти</button>
      </div>
    </form>

    {% if registration_open %}
      <p style="margin-top: 12px;">
        Нет аккаунта?
        <a href="{{ url_for('register') }}">Зарегистрируйтесь</a>.
      </p>
    {% endif %}
  </div>
{% endblock %}
```

#### `dashboard.html`

Переведём дашборд полностью на базовый шаблон:

```html
{% extends "base.html" %}

{% block title %}Дашборд · Учебное веб‑приложение{% endblock %}

{% block content %}
  <div class="card">
    <h2>Дашборд</h2>
    <p>Вы вошли как {{ user.username }} (роль: {{ user.role }}).</p>
    <p>Здесь находится основная функциональность приложения.</p>
  </div>
{% endblock %}
```

#### `admin_settings.html` и `register.html`

Шаблоны из шагов C1 и C2 также можно переписать через `extends "base.html"`, оставив в блоке `content` только собственную разметку формы. Структура `<html>`, `<head>`, `<body>` и навигация возьмутся из `base.html`.

**Проверка C3:**

1. Перезапустите приложение.
2. Откройте главную страницу, вход, регистрационную форму, дашборд и страницу настроек.
3. Убедитесь, что:
   - везде одинаковая шапка и навигация,
   - при входе/выходе ссылки в шапке меняются,
   - при выключенной регистрации пропадает ссылка «Регистрация».

---

## Что у вас есть после Части C

- Таблица `settings` с настройкой `registration_open` и функция `get_registration_open()` для чтения её значения.
- Страница `/admin/settings`, доступная только мастеру, с чекбоксом «Разрешить регистрацию новых пользователей».
- Страница `/register`, которая создаёт нового пользователя с безопасным паролем и учитывает состояние регистрации (открыта/закрыта).
- Базовый шаблон `base.html` с общей шапкой и навигацией; основные страницы (`home`, `login`, `dashboard`, настройки, регистрация) наследуют его через `{% extends "base.html" %}` и заполняют свой блок `{% block content %}`.

Дальше можно развивать основную предметную логику приложения, добавляя новые страницы и функции поверх уже готовой системы пользователей, сессий, регистрации и общего макета.
{% endraw %}

