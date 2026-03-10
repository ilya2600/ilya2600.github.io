{% raw %}
# Часть E: Разделяем `app.py` на несколько простых файлов

Цель этой части — по шагам **разбить большой файл `app.py` на несколько небольших модулей**, чтобы код было проще читать и сопровождать, **без сложных приёмов Flask и без blueprints**.

Мы будем использовать очень простой подход:

- `app.py` остаётся «центральной точкой входа»:
  - создаёт объект `app`,
  - содержит объявления маршрутов (`@app.route`, `@app.get`, `@app.post`),
  - внутри маршрутов просто вызывает функции из других файлов.
- Отдельные файлы содержат:
  - **вспомогательные функции для БД**,
  - **вспомогательные функции для пользователей и прав доступа**,
  - **функции‑обработчики логики страниц** (но **без** декораторов маршрутов).

---

## E1. План разбиения

Предлагаемая структура (минимальная и понятная):

- `app.py`  
  Создание `Flask`‑приложения, маршруты (`@app.get("/…")` и т.п.), подключение вспомогательных модулей.

- `db.py`  
  Функции работы с БД:
  - `DB_PATH` (при желании),
  - `get_conn()`,
  - `init_db()`,
  - `insert_test_user()`,
  - `show_table()`.

- `auth_utils.py`  
  Вспомогательные функции, связанные с пользователями и правами:
  - `create_user()`,
  - `ensure_master()`,
  - `is_logged_in()`,
  - `current_user()`,
  - `is_admin()`,
  - `get_registration_open()`.

- `views_auth.py`  
  Логика для маршрутов авторизации и аккаунта:
  - `login_form_view()`,
  - `login_view()`,
  - `logout_view()`,
  - `register_form_view()`,
  - `register_view()`,
  - `dashboard_view()`.

- `views_admin.py`  
  Логика админ‑страниц:
  - `admin_settings_view()`,
  - `admin_settings_save_view()`,
  - `admin_users_view()`,
  - `admin_user_create_view()`,
  - `admin_user_archive_view()`,
  - `admin_user_restore_view()`,
  - `admin_user_delete_view()`.


В этой части мы сделаем **минимальный** и понятный вариант, без усложнений.

---

## E2. Выносим функции работы с БД в `db.py`

### Шаг E2.1. Создаём файл `db.py`

1. В той же папке, где лежит `app.py`, создайте новый файл `db.py`.

### Шаг E2.2. Переносим функции

2. Из `app.py` **вырежьте** и перенесите в `db.py` полностью:

   - константу `DB_PATH` (если она у вас в `app.py`),
   - функцию `get_conn()`,
   - функцию `init_db()`,
   - функцию `insert_test_user()`,
   - функцию `show_table()`.

3. В начале `db.py` оставьте только необходимые импорты:

   - `import sqlite3`.

   Если какие‑то функции из этого файла используют ещё что‑то (например, дату или хеширование), импортируйте это тоже.

### Шаг E2.3. Обновляем `app.py`, чтобы использовать `db.py`

4. В начале `app.py` **уберите старые определения** `DB_PATH`, `get_conn`, `init_db`, `insert_test_user`, `show_table`.

5. В начале `app.py` добавьте импорт:

   ```python
   from db import get_conn, init_db, insert_test_user, show_table
   ```

6. Во всём файле `app.py` не нужно менять вызовы этих функций — они по‑прежнему будут вызываться просто как `get_conn()`, `init_db()` и т.д., только теперь реализованы в отдельном модуле.

---

## E3. Выносим функции пользователей и прав в `auth_utils.py`

Сейчас в `app.py` есть несколько функций, отвечающих за пользователей и права:

- `create_user(username, password, role)`,
- `ensure_master()`,
- `is_logged_in()`,
- `current_user()`,
- `is_admin()`,
- `get_registration_open()`.

Мы перенесём их в отдельный модуль.

### Шаг E3.1. Создаём файл `auth_utils.py`

1. Создайте новый файл `auth_utils.py` рядом с `app.py`.

### Шаг E3.2. Переносим функции

2. Из `app.py` **вырежьте** и перенесите в `auth_utils.py` полностью:

   - функцию `create_user(...)`,
   - функцию `ensure_master()`,
   - функцию `is_logged_in()`,
   - функцию `current_user()`,
   - функцию `is_admin()`,
   - функцию `get_registration_open()`.

3. В начале `auth_utils.py` добавьте нужные импорты:

   - `from flask import session`,
   - `from werkzeug.security import check_password_hash, generate_password_hash` (если используется хеширование),
   - `from datetime import datetime` (если нужно для архивирования),
   - `from db import get_conn`.

   Импортируйте именно то, что реально используется внутри этих функций.

### Шаг E3.3. Обновляем `app.py`, чтобы использовать `auth_utils`

4. В `app.py` удалите старые определения:

   - `create_user`,
   - `ensure_master`,
   - `is_logged_in`,
   - `current_user`,
   - `is_admin`,
   - `get_registration_open`.

5. В начале `app.py` добавьте импорт из `auth_utils.py`:

   ```python
   from auth_utils import (
       create_user,
       ensure_master,
       is_logged_in,
       current_user,
       is_admin,
       get_registration_open,
   )
   ```

6. Код маршрутов (`/login`, `/logout`, `/dashboard`, `/admin/...` и др.) можно **пока не трогать** — они будут пользоваться этими функциями так же, как раньше, просто теперь они приехали из модуля `auth_utils`.

---

## E4. Выносим логику авторизации в `views_auth.py`

Сейчас в `app.py` маршруты авторизации выглядят примерно так:

- `@app.get("/login") def login_form(): ...`
- `@app.post("/login") def login(): ...`
- `@app.get("/logout") def logout(): ...`
- `@app.get("/register") def register_form(): ...`
- `@app.post("/register") def register(): ...`
- `@app.get("/dashboard") def dashboard(): ...`

Каждый из них содержит и объявление маршрута, и саму логику. Мы хотим оставить **декораторы в `app.py`**, а **всю тяжёлую логику перенести в отдельный файл**. Например получится у нас в итоге так, вместо этого:

```python
@app.get("/login")
def login_form():
   ... старая логика ...
```

Будет вот это:

```python
@app.get("/login")
def login_form():
   return login_form_view()
```

А в отдельном файле `view_auth.py` мы создадим функции `view` и перенесем туда саму логику:

```python
def login_form_view():
   if is_logged_in():
         return redirect(url_for("home"))
      return render_template("login.html", error=None)
```

### Шаг E4.1. Создаём файл `views_auth.py`

1. Создайте файл `views_auth.py` рядом с `app.py`.

### Шаг E4.2. Создаём функции‑обработчики без декораторов

2. В `views_auth.py` создайте по одной пустой функции для каждого маршрута авторизации.  
   Например:

   - `login_form_view()` — для логики из старого `login_form`,
   - `login_view()` — для логики из старого `login`,
   - `logout_view()` — для логики из старого `logout`,
   - `register_form_view()` — для логики из старого `register_form`,
   - `register_view()` — для логики из старого `register`,
   - `dashboard_view()` — для логики из старого `dashboard`.

3. Внутрь этих функций **перенесите только тело** (то есть, саму логику) соответствующих маршрутов из `app.py`, без декораторов.  
   То есть:

   - в `login_form_view()` должно оказаться всё, что было внутри `login_form()` который под `@app.get("/login")`,
   - в `login_view()` — всё, что было внутри `login()`,
   - и так далее.

4. В начале `views_auth.py` добавьте необходимые импорты:

```python
from flask import render_template, request, redirect, url_for, session
from auth_utils import is_logged_in, current_user, get_registration_open
from db import get_conn
from werkzeug.security import check_password_hash, generate_password_hash
import sqlite3
```

### Шаг E4.3. Упрощаем маршруты в `app.py`

5. В `app.py` в начале файла добавьте:

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

6. Найдите старые определения маршрутов авторизации в `app.py` и **замените их на тонкие обёртки**. Например:

   - вместо

     ```python
     @app.get("/login")
     def login_form():
         ... старая логика ...
     ```

     сделайте:

     ```python
     @app.get("/login")
     def login_form():
         return login_form_view()
     ```

   - вместо

     ```python
     @app.post("/login")
     def login():
         ... старая логика ...
     ```

     сделайте:

     ```python
     @app.post("/login")
     def login():
         return login_view()
     ```

   И аналогично для:

   - `/logout` → `logout_view()`,
   - `/register` (GET) → `register_form_view()`,
   - `/register` (POST) → `register_view()`,
   - `/dashboard` → `dashboard_view()`.

7. В результате:

   - **вся логика** авторизации теперь лежит в `views_auth.py`,
   - `app.py` только объявляет маршруты и делегирует выполнение.

---

## E5. Выносим админ‑страницы в `views_admin.py`

Теперь сделаем то же самое для админ‑маршрутов:

- `@app.get("/admin/settings") def admin_settings(): ...`
- `@app.post("/admin/settings") def admin_settings_save(): ...`
- `@app.get("/admin/users") def admin_users(): ...`
- `@app.post("/admin/users/create") def admin_user_create(): ...`
- `@app.post("/admin/users/<int:user_id>/archive") def admin_user_archive(user_id): ...`
- `@app.post("/admin/users/<int:user_id>/restore") def admin_user_restore(user_id): ...`
- `@app.post("/admin/users/<int:user_id>/delete") def admin_user_delete(user_id): ...`

### Шаг E5.1. Создаём файл `views_admin.py`

1. Создайте файл `views_admin.py` рядом с `app.py`.

### Шаг E5.2. Переносим логику маршрутов

2. В `views_admin.py` создайте функции:

   - `admin_settings_view()`,
   - `admin_settings_save_view()`,
   - `admin_users_view()`,
   - `admin_user_create_view()`,
   - `admin_user_archive_view(user_id)`,
   - `admin_user_restore_view(user_id)`,
   - `admin_user_delete_view(user_id)`.

3. Внутрь каждой функции перенесите **тело** соответствующего маршрута из `app.py`:

   - так же как в предыдущем шаге с view функциями
   - без декоратора,
   - с сохранением имени аргументов (например, `user_id`).

4. В начале `views_admin.py` импортируйте всё нужное:

```python
from flask import render_template, request, redirect, url_for, abort, flash
from auth_utils import is_logged_in, is_admin, get_registration_open, create_user
from db import get_conn
from datetime import datetime
import sqlite3
```.

### Шаг E5.3. Упрощаем админ‑маршруты в `app.py`

5. В `app.py` добавьте импорт:

   ```python
   from views_admin import (
       admin_settings_view,
       admin_settings_save_view,
       admin_users_view,
       admin_user_create_view,
       admin_user_archive_view,
       admin_user_restore_view,
       admin_user_delete_view,
   )
   ```

6. Замените тела админ‑маршрутов на делегирование:

   - `/admin/settings` (GET):

     ```python
     @app.get("/admin/settings")
     def admin_settings():
         return admin_settings_view()
     ```

   - `/admin/settings` (POST):

     ```python
     @app.post("/admin/settings")
     def admin_settings_save():
         return admin_settings_save_view()
     ```

   - `/admin/users`:

     ```python
     @app.get("/admin/users")
     def admin_users():
         return admin_users_view()
     ```

   - остальные — по аналогии, вызывая соответствующие `*_view` функции и передавая `user_id`, если он есть.


## E6. Итоговая картина

После выполнения шагов этой части:

- **`db.py`** содержит функции работы с БД (`get_conn`, `init_db`, `insert_test_user`, `show_table` и, при желании, `DB_PATH`).
- **`auth_utils.py`** содержит логику пользователей и прав (`create_user`, `ensure_master`, `is_logged_in`, `current_user`, `is_admin`, `get_registration_open`).
- **`views_auth.py`**, **`views_admin.py`** содержат *логику* страниц (что делать при входе, регистрации, работе с пользователями и заявками), но **не** содержат `@app.route`.
- **`app.py`**:
  - создаёт `Flask`‑приложение,
  - подключает вспомогательные модули,
  - объявляет маршруты (`@app.get`, `@app.post`) и внутри вызывает функции из `views_*`.
  - подготавливает БД и начальные данные,
  - запускает приложение `app.run(debug=True)`.

Главная идея: **логика по темам разнесена по отдельным файлам**, а `app.py` превратился в «карту маршрутов» и точку входа.  
Такой подход упрощает чтение кода и почти не меняет способ работы с Flask, поэтому хорошо подходит как первый шаг к более модульной структуре проекта.
{% endraw %}

