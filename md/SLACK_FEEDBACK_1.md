# Руководство по исправлению критичных ошибок (ядро функциональности)

Этот документ помогает исправить ошибки на сервере, которые **ломают основную работу приложения** (регистрация, переходы по страницам, удаление сообщений и т.п.).


---

## 0) Перед началом

1) Установите зависимости Python:

```powershell
pip install -r requirements.txt
```

2) После каждого изменения кода **перезапускайте сервер**.

3) Если в браузере вы видите ошибку, обязательно смотрите **текст ошибки в терминале**, где запущен Flask — там почти всегда указаны файл и строка, где всё упало.

---

## 1) Регистрация сломана (несовпадение схемы БД + неправильная роль + отсутствует импорт `sqlite3`)

### В чём проблема

Регистрация происходит в функции `register_view()` в файле `views_auth.py`.

Схема таблицы `users` задана в `db.py`. В ней:

- колонка называется `password` (а **не** `password_hash`)
- допустимые роли: только `user` или `admin` (а **не** `agent`)

Но текущий код регистрации пытается вставить:

- `password_hash` (такой колонки в таблице **нет**)
- роль `'agent'` (ограничение роли в БД это **запрещает**)
- и в обработчике ошибок используется `sqlite3.IntegrityError`, но `sqlite3` **не импортирован**

В итоге регистрация падает с ошибками во время выполнения.

### Как исправить (копировать/вставить)

#### Шаг A: Добавьте импорт

Откройте `views_auth.py` и добавьте рядом с другими импортами:

```python
import sqlite3
```

Важно: импорт должен быть **вверху файла**, а не внутри функции.

#### Шаг B: Исправьте SQL INSERT

В `views_auth.py` внутри `register_view()` найдите этот кусок:

```python
conn.execute(
    "INSERT INTO users (username, password_hash, role) VALUES (?, ?, 'agent')",
    (username, generate_password_hash(password)),
)
```

Замените на этот (правильные колонки + правильная роль):

```python
conn.execute(
    "INSERT INTO users (username, password, role) VALUES (?, ?, 'user')",
    (username, generate_password_hash(password)),
)
```

#### Шаг C: Убедитесь, что обработчик ошибок остался корректным

Проверьте, что `try/except` выглядит примерно так:

```python
except sqlite3.IntegrityError:
    conn.close()
    return render_template("register.html", error="Username already taken."), 409
```

### После исправления

1) Перезапустите сервер.
2) Откройте `/register`.
3) Создайте нового пользователя.
4) Попробуйте войти под созданным пользователем.

---

## 2) Сломанная ссылка на дашборд в `templates/index.html` (неверный `url_for('/')`)

### В чём проблема

Файл: `templates/index.html`

Там есть строка:

```html
<a href="{{ url_for('/') }}">Дашборд каналов</a>
```

Функция `url_for()` принимает **имя endpoint-а** (например `dashboard`), а не строку пути `"/"`.

Если шаблон `index.html` когда-нибудь отрендерится — приложение упадёт с ошибкой маршрутизации.

Дополнительно: для гостя там стоит `url_for('login')`, но `login` — это **POST-роут** (обработчик отправки формы). Для ссылки нужен GET-роут формы входа: `login_form`.

### Как исправить (копировать/вставить)

Откройте `templates/index.html` и замените:

```html
<a href="{{ url_for('/') }}">Дашборд каналов</a> ·
```

на:

```html
<a href="{{ url_for('dashboard') }}">Дашборд каналов</a> ·
```

Дальше замените:

```html
<a href="{{ url_for('login') }}">Войти</a>
```

на:

```html
<a href="{{ url_for('login_form') }}">Войти</a>
```

### После исправления

Перезапустите Flask и проверьте, что ссылки на странице работают и не вызывают ошибок.

---

## 3) Дублирование `message_delete_view` (вторая функция перезаписывает первую)

### В чём проблема

Файл: `views_channels.py`

Функция `message_delete_view(channel_id, message_id)` объявлена **два раза**.

В Python это означает: **вторая версия молча перезапишет первую**. Это приводит к очень странным багам и ломает поддержку проекта.

### Как исправить (копировать/вставить)

Откройте `views_channels.py` и найдите:

```python
def message_delete_view(channel_id: int, message_id: int):
```

Вы увидите **две** одинаковые функции.

Сделайте так:

1) Оставьте **только одну** реализацию `message_delete_view`.
2) Вторую реализацию удалите целиком.
3) Проверьте, что после оставшейся функции следующей идёт `def reply_create_view(...)`.

### После исправления

Перезапустите Flask и протестируйте удаление треда/ответа через кнопки в интерфейсе.

---

## 4) Исправление "простых" учётных данных администратора `master` через `.env`

### Зачем это менять (в рамках учебного проекта)

Сейчас код автоматически создаёт администратора с логином/паролем `master/master`.

Чтобы приложение работало и при этом не было "угадываемого" админа, сделаем так:

1) Добавим файл `.env` с `MASTER_USERNAME` и `MASTER_PASSWORD`
2) На старте приложения загрузим `.env`
3) В `ensure_master()` будем создавать админа **только** из значений `.env`

### Шаг A: Создайте файл `.env`

В корне репозитория (там же, где `app.py`) создайте файл:

`C:\Users\Admin\Downloads\slackchat-main_2\.env`

Содержимое:

```env
MASTER_USERNAME=master
MASTER_PASSWORD=PUT_A_STRONG_PASSWORD_HERE

# Опционально: включить/выключить создание тестовых пользователей при запуске.
DEV_SEED_DATA=0
```

Важно: замените `PUT_A_STRONG_PASSWORD_HERE` на пароль, который вы выбрали.

### Шаг B: Установите `python-dotenv`

В PowerShell в папке проекта выполните:

```powershell
pip install python-dotenv
```

Затем добавьте строку в `requirements.txt` (можно в конец файла):

```txt
python-dotenv
```

И на всякий случай переустановите зависимости:

```powershell
pip install -r requirements.txt
```

### Шаг C: Загружайте `.env` в `app.py`

Откройте `app.py` и добавьте импорты (рядом с другими импортами):

```python
import os
from dotenv import load_dotenv

load_dotenv()
```

Далее исправьте блок запуска внизу `app.py`.

Найдите текущий код:

```python
if __name__ == "__main__":
    init_db()
    insert_test_user()
    print(show_table())
    ensure_master()
    create_user("test1","123","user")
    app.run(debug=True)
```

И замените на:

```python
if __name__ == "__main__":
    init_db()
    print(show_table())
    ensure_master()

    if os.getenv("DEV_SEED_DATA", "0") == "1":
        insert_test_user()
        create_user("test1", "123", "user")

    app.run(debug=True)
```

Пояснение:

- `ensure_master()` создаст администратора из значений `.env`
- `DEV_SEED_DATA=0` (по умолчанию) означает, что слабые тестовые пользователи не будут создаваться автоматически

### Шаг D: Сделайте `ensure_master()` зависимым от `.env`

Откройте `auth_utils.py` и добавьте импорт:

```python
import os
```

Далее в `auth_utils.py` найдите в `ensure_master()` этот фрагмент:

```python
if row is None:
    create_user("master", "master", "admin")
```

Замените на:

```python
if row is None:
    master_username = os.getenv("MASTER_USERNAME")
    master_password = os.getenv("MASTER_PASSWORD")

    if not master_username or not master_password:
        raise RuntimeError("В .env отсутствует MASTER_USERNAME и/или MASTER_PASSWORD")

    create_user(master_username, master_password, "admin")
```

### После исправления

1) Запустите сервер.
2) Войдите под админом:
   - логин = значение `MASTER_USERNAME` из `.env`
   - пароль = значение `MASTER_PASSWORD` из `.env`
3) Проверьте, что регистрация/вход/каналы работают как раньше.

---

## 5) (Опционально) Если падает шаблон `channel_view.html` из-за `current_user().id`

### Что может быть не так

Файл: `templates/channel_view.html`

Там есть условие вида:

```html
current_user().id
```

Обычно `current_user()` возвращает `sqlite3.Row`. В некоторых случаях Jinja может неудачно обрабатывать доступ через точку `.id`.

### Как исправить (копировать/вставить)

В `templates/channel_view.html` замените:

```html
r.author_id == current_user().id or channel.owner_id == current_user().id
```

на:

```html
r.author_id == current_user()['id'] or channel.owner_id == current_user()['id']
```

После этого перезапустите Flask.

---

## Если после этого всё ещё падает

1) Скопируйте полный traceback из терминала (ошибка Python).
2) Напишите, что именно вы делали (регистрация, вход, удаление сообщения, создание канала и т.д.).

По этим двум пунктам обычно можно за 1-2 шага найти оставшуюся строку, которая ломает приложение.

