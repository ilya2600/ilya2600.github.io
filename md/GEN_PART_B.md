{% raw %}
# Часть B: Пользователи и вход в систему

Цель этой части — по шагам добавить в приложение:

- учётную запись мастера (администратора),
- безопасное хранение паролей (хеши, а не «голые» строки),
- форму входа и проверку пароля,
- сессию (запоминать, кто вошёл),
- выход из системы,
- простую защищённую страницу‑дашборд.

Мы продолжаем использовать те же функции и подходы, что в Части A:

- `get_conn()` — подключение к SQLite;
- таблица `users` с полями `username`, `password`, `role`, `archived_at`, `created_at`;
- вспомогательные функции `insert_test_user()` и `show_table()`.

---

## B1. Учётная запись мастера и функция `create_user`

Сначала сделаем так, чтобы в системе **обязательно был один административный аккаунт**. Заодно введём универсальную функцию для создания пользователей — потом мы её доработаем под хеширование паролей.

### 1. Общая функция `create_user`

В `app.py`, **рядом с функциями работы с БД** (`get_conn`, `init_db`, `insert_test_user`, `show_table`), добавьте новую функцию:

```python
def create_user(username, password, role):
    """
    Создать пользователя с указанным логином, паролем и ролью.
    На этом этапе пароль сохраняется как есть (будем улучшать позже).
    """
    conn = get_conn()
    conn.execute(
        "INSERT INTO users (username, password, role) VALUES (?, ?, ?)",
        (username, password, role),
    )
    conn.commit()
    conn.close()
```

Пока что:

- `password` попадает в столбец `password` как обычная строка;
- позже мы заменим это на хеширование и не придётся переписывать все места создания пользователей — достаточно изменить реализацию `create_user`.

### 2. Функция `ensure_master`

Теперь добавим функцию, которая **гарантирует, что в таблице есть хотя бы один администратор**. Если его нет — создаём «мастер‑аккаунт».

```python
def ensure_master():
    """
    Убедиться, что в системе есть хотя бы один администратор.
    Если нет — создать пользователя master/master с ролью admin.
    """
    conn = get_conn()
    row = conn.execute(
        "SELECT id FROM users WHERE role = 'admin' AND archived_at IS NULL LIMIT 1"
    ).fetchone()

    if row is None:
        create_user("master", "master", "admin")

    conn.close()
```

Логика проста:

- пробуем найти любого живого (`archived_at IS NULL`) пользователя с ролью `admin`;
- если не нашли — вызываем `create_user("master", "master", "admin")`;
- таким образом при первом запуске создаётся мастер‑аккаунт, а при последующих — ничего лишнего не добавляется.

### 3. Обновляем блок запуска приложения

В конце `app.py` найдите блок:

```python
if __name__ == "__main__":
    init_db()
    # insert_test_user()
    # print(show_table())
    app.run(debug=True)
```

и замените его на вариант, который инициализирует базу и гарантирует наличие мастера:

```python
if __name__ == "__main__":
    init_db()
    ensure_master()
    # при необходимости можно вызвать insert_test_user() или print(show_table())
    app.run(debug=True)
```

### 4. Проверка шага B1

1. Удалите старый файл базы (`database.db`), если хотите начать с нуля.
2. Запустите приложение: `python app.py`.
3. После запуска (или временно, перед `app.run`) вы можете напечатать:

   ```python
   print(show_table())
   ```

   и убедиться, что в таблице `users` есть пользователь:

   - `username = "master"`,
   - роль `admin`,
   - пароль пока хранится как строка `"master"`.

На этом шаге мы **сознательно** храним пароль в открытом виде — это временно, следующий шаг исправит это.

---

## B2. Безопасное хранение паролей (хеширование)

Хранить пароль как строку в базе нельзя — если база утечёт, злоумышленник сразу увидит все пароли. Вместо этого храним только **хеш** пароля — результат односторонней функции.

Flask‑проекты обычно используют функции из `werkzeug.security`:

- `generate_password_hash(plain_password)` — принимает исходный пароль и возвращает строку‑хеш;
- `check_password_hash(stored_hash, plain_password)` — сравнивает сохранённый хеш и введённый пользователем пароль.

### 1. Импорт функций хеширования

В начале `app.py`, рядом с другими импортами, добавьте:

```python
from werkzeug.security import check_password_hash, generate_password_hash
```

### 2. Обновляем `create_user` под хеширование

Вернитесь к функции `create_user` и замените её содержимое на вариант с хешем:

```python
def create_user(username, password, role):
    """
    Создать пользователя: пароль сразу превращаем в безопасный хеш.
    В таблицу users никогда не попадает «голый» пароль.
    """
    password_hash = generate_password_hash(password)

    conn = get_conn()
    conn.execute(
        "INSERT INTO users (username, password, role) VALUES (?, ?, ?)",
        (username, password_hash, role),
    )
    conn.commit()
    conn.close()
```

Обратите внимание:

- теперь в столбец `password` мы записываем результат `generate_password_hash(password)`;
- остальной код, который вызывает `create_user`, можно не менять — поведение улучшилось «под капотом».

Функцию `ensure_master()` **не нужно менять**:

- она по‑прежнему вызывает `create_user("master", "master", "admin")`;
- внутри `create_user` уже используется `generate_password_hash`, значит мастер‑аккаунт тоже хранится безопасно.

### 4. HTML‑форма входа `login.html`

Теперь добавим простую страницу входа:

- поле `username`;
- поле `password`;
- возможность отобразить сообщение об ошибке.

В папке `templates/` создайте файл `login.html` со следующим содержимым:

```html
<!DOCTYPE html>
<html lang="ru">
  <head>
    <meta charset="utf-8" />
    <title>Вход</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='styles.css') }}" />
  </head>
  <body>
    <div class="card">
      <h1>Вход в систему</h1>

      {% if error %}
        <p style="color: darkred;">{{ error }}</p>
      {% endif %}

      <form method="post" action="{{ url_for('login') }}">
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
    </div>
  </body>
</html>
```

Здесь используется:

- `url_for('static', filename='styles.css')` — тот же CSS, что и на главной странице;
- `url_for('login')` — путь к обработчику POST‑запроса входа (мы его сейчас создадим);
- переменная `error` — опциональное сообщение об ошибке, которое передаётся из обработчика.

### 5. Маршруты `/login` (форма и обработка)

В `app.py` подключите модули Flask, которые нужны для форм и редиректов, если их ещё нет:

```python
from flask import ..., request, redirect, url_for
```

Теперь добавим два маршрута:

- `GET /login` — показать форму входа, в коде как `@app.get(...)`;
- `POST /login` — обработать отправку формы, проверить пароль, в коде как `@app.post(...)`.

```python
@app.get("/login")
def login_form():
    # На этом шаге сессией ещё не пользуемся — просто показываем форму
    return render_template("login.html", error=None)


@app.post("/login")
def login():
    username = request.form.get("username", "").strip()
    password = request.form.get("password", "")

    if not username or not password:
        return render_template(
            "login.html",
            error="Введите логин и пароль.",
        )

    conn = get_conn()
    user = conn.execute(
        "SELECT * FROM users WHERE username = ? AND archived_at IS NULL LIMIT 1",
        (username,),
    ).fetchone()
    conn.close()

    if user is None:
        return render_template(
            "login.html",
            error="Пользователь не найден.",
        )

    if not check_password_hash(user["password"], password):
        return render_template(
            "login.html",
            error="Неверный пароль.",
        )

    # Пока что просто покажем заглушку (без сессии)
    return render_template("home.html", db_ok=True)
```

Здесь важно:

- если пользователь не найден — показываем ошибку;
- если пароль не совпадает — тоже ошибка;
- при успехе пока **не запоминаем** пользователя, просто рендерим страницу (сессию добавим в следующем шаге).

На практике вместо `render_template("home.html", db_ok=True)` вы можете:

- либо аккуратно подставить реальные данные (`db_ok` как в Части A),
- либо временно создать простой шаблон‑заглушку (`login_success.html`).

Главная идея этого шага — показать использование `check_password_hash`.

---

## B3. Сессия: запомнить, кто вошёл

В `app.py` подключите модуль session:

```python
from flask import ..., session
```

Сейчас после успешного входа приложение «забывает» пользователя при каждом новом запросе. Нам нужно:

- **записать идентификатор пользователя в сессию** после логина;
- **читать его из сессии** при обработке других маршрутов.

Сессия в Flask хранится в объекте `session` (словарь, связанный с текущим пользователем). Для работы с сессией у приложения должен быть установлен секретный ключ.

### 1. Секретный ключ и импорт `session`

В начале файла `app.py` убедитесь, что:

- импортирован `session` из Flask (мы уже добавили его выше);
- у приложения задан `secret_key`:

```python
app = Flask(__name__)
app.secret_key = "dev-secret"  # в реальном проекте использовать надёжный случайный ключ
```

### 2. Функция `is_logged_in`

Добавим простую проверку, вошёл ли пользователь:

```python
def is_logged_in():
    return session.get("user_id") is not None
```

Эта функция:

- возвращает `True`, если в сессии есть ключ `user_id`;
- иначе — `False`.

### 3. Обновляем обработчик `login` под сессию

Теперь дополним функцию `login`, чтобы при успешной проверке пароля:

- очистить сессию;
- сохранить в неё `user_id` и `role`;
- сделать редирект.

Замените «успешную» часть функции `login` на такую:

```python
    ...
    if not check_password_hash(user["password"], password):
        return render_template(
            "login.html",
            error="Неверный пароль.",
        )

    # Пароль подошёл — запоминаем пользователя в сессии
    session.clear()
    session["user_id"] = user["id"]
    session["role"] = user["role"]

    # На этом шаге можно просто отправить на главную
    return redirect(url_for("home"))
```

Теперь после успешного входа:

- браузер получает редирект на маршрут `home`;
- в сессии уже записан `user_id`, и мы можем использовать это дальше.

### 4. Улучшаем `login_form`: не показывать форму, если пользователь уже вошёл

Обновим функцию `login_form`:

```python
@app.get("/login")
def login_form():
    if is_logged_in():
        return redirect(url_for("home"))

    return render_template("login.html", error=None)
```

Логика:

- если в сессии уже есть пользователь — нет смысла делать вид, что он «не вошёл»;
- сразу отправляем на главную страницу;
- иначе — показываем форму входа.

---

## B4. Выход из системы

Нужна возможность «забыть» пользователя и очистить сессию.

### 1. Маршрут `/logout`

В `app.py` добавьте:

```python
@app.get("/logout")
def logout():
    session.clear()
    return redirect(url_for("home"))
```

Здесь:

- `session.clear()` удаляет все данные из сессии (`user_id`, `role` и т.п.);
- затем браузер перенаправляется на главную.

### 2. Ссылки на выход в шаблонах

В тех шаблонах, где уместно (например, в `home.html` и чуть позже в `dashboard.html`), можно добавить ссылку:

```html
<a href="{{ url_for('logout') }}">Выйти</a>
```

Дальше мы покажем, как делать это условно — только если пользователь вошёл.

---

## B5. `current_user` и разный контент на главной

Сейчас мы знаем в сессии только `user_id`, но при каждом рендеринге страниц было бы удобно иметь доступ к «текущему пользователю» целиком.

Сделаем вспомогательную функцию `current_user()` и научимся показывать разный контент на главной странице:

- если пользователь не вошёл — предлагаем войти;
- если вошёл — показываем его имя и ссылку на выход / дашборд.

### 1. Функция `current_user`

В `app.py` добавьте:

```python
def current_user():
    user_id = session.get("user_id")
    if user_id is None:
        return None

    conn = get_conn()
    user = conn.execute(
        "SELECT * FROM users WHERE id = ? AND archived_at IS NULL",
        (user_id,),
    ).fetchone()
    conn.close()

    return user
```

Поведение:

- если в сессии нет `user_id` — функция возвращает `None`;
- если есть — читает пользователя из таблицы `users` (мёртвых / архивных игнорируем);
- возвращает либо строку (объект `sqlite3.Row`), либо `None`, если запись не найдена.

### 2. Передаём пользователя в шаблон главной страницы

Найдите маршрут `home` из Части A (он делает проверочный запрос `SELECT 1 AS ok`) и обновите его так, чтобы он также передавал в шаблон текущего пользователя:

```python
@app.route("/")
def home():
    conn = get_conn()
    row = conn.execute("SELECT 1 AS ok").fetchone()
    conn.close()

    db_ok = row is not None and row["ok"] == 1
    user = current_user()

    return render_template("home.html", db_ok=db_ok, user=user)
```

### 3. Обновляем `home.html`: разный контент

Откройте `templates/home.html` и внутри основного блока (например, внутри `<div class="card">`) добавьте условный фрагмент:

```html
<!-- например, внутри `<div class="card">` -->
{% if user %}
  <p>Вы вошли как <strong>{{ user.username }}</strong>.</p>
  <p>
    <a href="{{ url_for('logout') }}">Выйти</a>
    |
    <a href="{{ url_for('dashboard') }}">Перейти в дашборд</a>
  </p>
{% else %}
  <p>Вы не вошли в систему.</p>
  <p>
    <a href="{{ url_for('login_form') }}">Войти</a>
  </p>
{% endif %}
```

Здесь используются:

- `user.username` — поле пользователя из БД;
- `url_for('logout')` — ссылка на маршрут выхода;
- `url_for('login_form')` — ссылка на форму входа;
- `url_for('dashboard')` — ссылка на будущий дашборд (создадим его в следующем шаге).

---

## B6. Простая защищённая страница‑дашборд

Осталось сделать отдельную страницу, которая:

- доступна только авторизованным пользователям;
- редиректит неавторизованных на форму входа;
- показывает минимальный контент, зависящий от пользователя.

### 1. Маршрут `/dashboard`

В `app.py` добавьте:

```python
from flask import abort, request  # если ещё не импортированы
```

и ниже других маршрутов — защищённый дашборд:

```python
@app.get("/dashboard")
def dashboard():
    if not is_logged_in():
        # next=request.url — чтобы при желании можно было потом вернуть пользователя обратно
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    if user is None:
        # На всякий случай: если user_id в сессии битый
        session.clear()
        return redirect(url_for("login_form"))

    return render_template("dashboard.html", user=user)
```

Здесь:

- мы переиспользуем `is_logged_in()` и `current_user()`;
- при отсутствии логина делаем редирект на `login_form`;
- передаём пользователя в шаблон.

### 2. Шаблон `dashboard.html`

В папке `templates/` создайте файл `dashboard.html`:

```html
<!DOCTYPE html>
<html lang="ru">
  <head>
    <meta charset="utf-8" />
    <title>Дашборд</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='styles.css') }}" />
  </head>
  <body>
    <div class="card">
      <h1>Дашборд</h1>
      <p>Здравствуйте, <strong>{{ user.username }}</strong>.</p>

      <p>
        <a href="{{ url_for('home') }}">На главную</a>
        |
        <a href="{{ url_for('logout') }}">Выйти</a>
      </p>

      <p>Здесь в будущем появится основная функциональность приложения.</p>
    </div>
  </body>
</html>
```

Пока это просто аккуратная заглушка.

### 3. После логина отправляем сразу в дашборд

Вернитесь к функции `login` и обновите финальный редирект:

```python
    ...
    session.clear()
    session["user_id"] = user["id"]
    session["role"] = user["role"]

    # Теперь после входа отправляем пользователя в дашборд
    return redirect(url_for("dashboard"))
```

Также убедитесь, что в `home.html` ссылка на дашборд уже использует `url_for('dashboard')` (как в примере выше).

### 4. Проверка всей связки

1. Убедитесь, что приложение запущено (`python app.py`).
2. Откройте главную страницу:
   - если вы ещё не логинились — увидите вариант «Вы не вошли в систему» с ссылкой на `/login`;
   - если в сессии уже есть пользователь — увидите его имя и ссылку в дашборд.
3. Перейдите на `/login`:
   - войдите под `master` / `master` или под `testuser` / `testpass` (если вызывали `insert_test_user()` после перехода на хеши);
   - после успешного входа произойдёт редирект в `/dashboard`.
4. Попробуйте открыть `/dashboard` в отдельном «чистом» браузере / вкладке (без сессии):
   - вы должны оказаться на странице входа;
   - после логина — снова в дашборде.
5. Нажмите «Выйти»:
   - сессия очистится;
   - вы вернётесь на главную в состоянии «не вошёл».

---

**Итог Части B:**

- есть универсальная функция `create_user` и инициализация `ensure_master`, которая гарантирует наличие административного аккаунта;
- пароли всех новых пользователей хранятся в виде хешей через `generate_password_hash`;
- реализованы форма входа и проверка пароля через `check_password_hash`;
- используется сессия Flask (`session`), функции `is_logged_in()` и `current_user()` для работы с текущим пользователем;
- есть выход из системы (`/logout`);
- добавлен простой защищённый дашборд (`/dashboard`), доступный только после входа.

На этой основе в следующих частях можно добавлять более сложные роли, дополнительные страницы и основную предметную логику приложения.
{% endraw %}

