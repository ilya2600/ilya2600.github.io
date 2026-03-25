# Исправление критичных ошибок (LMS / Flask + SQLite)

Ниже — список проблем, которые **ломают работу приложения** или приводят к ошибкам во время выполнения, и точные шаги, как это исправить.

---

## Что подготовить

### 1) Создайте и активируйте виртуальное окружение (Windows PowerShell)

В корне проекта:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

### 2) Установите зависимости

```powershell
pip install -r requirements.txt
```

---

## Проблема №1. Роли пользователей: база запрещает `user`, а код её использует (из-за этого приложение падает)

### Симптомы

- При запуске `python app.py` приложение может упасть с ошибкой SQLite наподобие:
  - `sqlite3.IntegrityError: CHECK constraint failed: ...`
- Или при создании пользователя из админки может возникать ошибка вставки в таблицу `users`.

### Причина

В `db.py` в таблице `users` роль ограничена (CHECK constraint) только такими значениями:

- `student`
- `teacher`
- `manager`
- `admin`

Но в проекте встречается роль **`user`**, которую база **не принимает**.

### Что именно исправить

#### Вариант A (рекомендуется): полностью убрать роль `user` и использовать корректные роли

##### Шаг A1. Исправьте тестового пользователя в `db.py`

Откройте файл `db.py` и найдите функцию `insert_test_user()`.

Замените строку вставки **с `'user'` на `'student'`**.

**Было:**

```python
conn.execute(
    "INSERT OR IGNORE INTO users (username, password, role) VALUES (?, ?, 'user')",
    ("testuser", "placeholder_hash"),
)
```

**Стало:**

```python
conn.execute(
    "INSERT OR IGNORE INTO users (username, password, role) VALUES (?, ?, 'student')",
    ("testuser", "placeholder_hash"),
)
```

##### Шаг A2. Исправьте создание пользователя в админке (`views_admin.py`)

Откройте `views_admin.py` и найдите функцию `admin_user_create_view()`.

Там уже вычисляется переменная `role` (из формы) и проверяется `allowed_roles`, но затем вызывается:

```python
create_user(username, password, "user")
```

Это **неправильно**, потому что:

- `"user"` запрещён базой;
- выбранная роль из формы игнорируется.

Замените на:

```python
create_user(username, password, role)
```

Дополнительно исправьте сообщение `flash`, чтобы подставлялось имя пользователя.

**Было:**

```python
create_user(username, password, "user")
flash("Пользователь {username} создан.")
```

**Стало:**

```python
create_user(username, password, role)
flash(f"Пользователь {username} создан.")
```

##### Шаг A3. (Опционально) Не создавать тестового пользователя автоматически при запуске

В `app.py` в блоке:

```python
if __name__ == "__main__":
    init_db()
    ensure_master()
    insert_test_user()
    ...
```

Если тестовый пользователь вам не нужен — удалите вызов `insert_test_user()` целиком (или оставьте, но после исправления роли он перестанет ломать запуск).

---

## Проблема №2. Архивация пользователя из админки падает из-за неправильного SELECT (нет поля `role`)

### Симптомы

При попытке заархивировать пользователя (эндпоинт `POST /admin/users/<id>/archive`) приложение может упасть с ошибкой доступа к полю, например:

- `IndexError: No item with that key`
- или похожая ошибка при `u["role"]`

### Причина

В `views_admin.py` в `admin_user_archive_view()` выбирается только `id`:

```python
u = conn.execute("SELECT id FROM users WHERE id = ?", (user_id,)).fetchone()
```

Но ниже код читает `u["role"]`, которого в выборке нет.

### Как исправить

В `views_admin.py`, внутри `admin_user_archive_view()`, замените запрос `SELECT id ...` на запрос, который выбирает **role**.

**Было:**

```python
u = conn.execute("SELECT id FROM users WHERE id = ?", (user_id,)).fetchone()
```

**Стало (минимальный фикс):**

```python
u = conn.execute("SELECT id, role FROM users WHERE id = ?", (user_id,)).fetchone()
```

После этого проверка:

```python
if u["role"] == "admin":
```

будет работать корректно.

---

## Проблема №3. Дублирующийся `return` в логине (не ломает всё, но это ошибка в логике)

### Симптомы

В `views_auth.py` в `login_view()` есть два `return` подряд:

```python
return redirect(url_for("home"))
return redirect(url_for("dashboard"))
```

Второй `return` **никогда не выполнится**.

### Как исправить

Решите, куда вы хотите перенаправлять после успешного входа (обычно — на `dashboard`).

**Вариант (перенаправлять на dashboard):**

Замените участок на:

```python
return redirect(url_for("dashboard"))
```

**Вариант (перенаправлять на главную):**

Оставьте:

```python
return redirect(url_for("home"))
```

и удалите второй `return`.

---

## Проблема №4. Мастер-аккаунт создаётся с простыми учётными данными — исправление через `.env`

Здесь цель — чтобы логин/пароль мастера (и секретный ключ) **не были захардкожены в коде** и задавались через файл `.env`.

### Шаг 4.1. Добавьте зависимость для чтения `.env`

Самый простой способ — пакет `python-dotenv`.

1) Откройте `requirements.txt` и добавьте строку:

```text
python-dotenv==1.1.1
```

2) Установите зависимости заново:

```powershell
pip install -r requirements.txt
```

### Шаг 4.2. Создайте файл `.env` в корне проекта

Создайте файл `.env` рядом с `app.py` и вставьте (значения поменяйте на свои):

```env
FLASK_SECRET_KEY=change-me-to-a-long-random-string

MASTER_USERNAME=master
MASTER_PASSWORD=change-me-now
MASTER_ROLE=admin
```

Рекомендации:
- `FLASK_SECRET_KEY` должен быть длинным и случайным.
- `MASTER_PASSWORD` обязательно измените.
- `MASTER_ROLE` оставьте `admin` (в этой базе другие роли для “мастера” обычно не нужны).

### Шаг 4.3. Загрузите `.env` в `app.py` до создания приложения и до `ensure_master()`

Откройте `app.py`.

1) Добавьте импорт:

```python
from dotenv import load_dotenv
import os
```

2) В самом начале файла (после импортов) вызовите `load_dotenv()`:

```python
load_dotenv()
```

Важно: это должно выполняться **до** того, как вы читаете переменные окружения.

### Шаг 4.4. Используйте `.env` в `app.py` для `secret_key`

Найдите:

```python
app.secret_key = "dev-secret"
```

Замените на:

```python
app.secret_key = os.getenv("FLASK_SECRET_KEY", "dev-secret")
```

(Запасное значение `"dev-secret"` оставлено как fallback, чтобы приложение не падало, если `.env` забыли создать. Если хотите строгий режим — можно вместо fallback делать `raise`.)

### Шаг 4.5. Переделайте `ensure_master()` так, чтобы он брал логин/пароль из окружения

Откройте `auth_utils.py` и найдите функцию `ensure_master()`.

1) Добавьте импорт `os` (если его нет):

```python
import os
```

2) Замените строку, где создаётся мастер:

**Было:**

```python
create_user("master", "master", "admin")
```

**Стало:**

```python
username = os.getenv("MASTER_USERNAME", "master")
password = os.getenv("MASTER_PASSWORD", "master")
role = os.getenv("MASTER_ROLE", "admin")
create_user(username, password, role)
```

После этого мастер будет создаваться с логином/паролем из `.env`.

### Шаг 4.6. Добавьте `.env` в `.gitignore` (если вы используете git)

Если появится файл `.gitignore` — добавьте строку:

```gitignore
.env
```

Это защищает от случайной публикации ваших паролей.

---

## Проверка после исправлений

1) Удалите старую базу (если она уже создана с ошибками) — файл `database.db` в корне проекта.
   - Важно: удаление базы удалит данные.

2) Запустите приложение:

```powershell
python app.py
```

3) Откройте в браузере:
- `/` (главная)
- `/login` (вход)
- `/admin/users` (список пользователей — после входа админом)

Если после этих шагов всё стартует без `IntegrityError` и без ошибок на архивации пользователя — критичные проблемы исправлены.

