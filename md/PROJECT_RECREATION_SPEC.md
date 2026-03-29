# MiniSocial — спецификация для воссоздания проекта

Документ описывает **всё, что делает эталонная реализация**, чтобы вы могли собрать то же приложение. Названия стека, лимиты, URL и правила данных соответствуют коду в `minisocial/` и шаблонам.

---

## 1. Назначение и охват

**MiniSocial** — небольшое веб-приложение в духе Twitter: пользователи входят в систему, читают ленту **по времени** (**newest**) или **в тренде** (**trending**), создают текстовые посты с опциональной **галереей изображений** (загрузка файла, URL или рисунок пиксель-арта 8×8), ставят **лайки**, задают **аватар PNG 8×8** через пиксель-редактор, **администраторы** управляют пользователями и регистрацией. Данные хранятся в **SQLite**, сессии — на сервере (Flask `session`).

---

## 2. Стек и зависимости

| Компонент | Детали |
|-----------|--------|
| Среда | Python 3.14.x (версия зафиксирована в `.python-version`; курс может уточнять отдельно) |
| Веб | Flask 3.x (фабрика `create_app` в `minisocial/__init__.py`) |
| Конфиг | `python-dotenv` — загрузка `.env` из корня проекта |
| Пароли | `werkzeug.security.generate_password_hash` / `check_password_hash` |
| Деплой | `gunicorn` (опционально; та же `create_app`) |
| БД | стандартный `sqlite3`; `PRAGMA foreign_keys = ON`; фабрика `Row` |

**Фронтенд:** обычный HTML (Jinja2), CSS (`static/style.css`), ванильный JS (`static/app.js`, `static/pixel-editor.js`), **emoji-picker-element** подключается как ESM с jsDelivr (`static/emoji-wc.js`).

---

## 3. Структура проекта (роли файлов)

| Путь | Назначение |
|------|------------|
| `app.py` | Импортирует `create_app()` из `minisocial`; при `if __name__ == "__main__"` вызывает `app.run(debug=True)`. |
| `minisocial/__init__.py` | Создаёт приложение Flask, задаёт `template_folder` / `static_folder` на `templates/` и `static/`, подключает конфиг, регистрирует context processors и маршруты, в контексте приложения вызывает `init_db()`, при необходимости — демо-заполнение. |
| `minisocial/config.py` | Константы, загрузка `.env`, `build_flask_config()`, проверки для production. |
| `minisocial/db.py` | Подключение к БД, схема, миграции, настройки, проверка размеров PNG, `init_db()`. |
| `minisocial/auth.py` | Декораторы `login_required`, `admin_or_master_required`, `master_required`; `current_user_can_manage_post`. |
| `minisocial/context.py` | Пробрасывает в шаблоны лимиты постов и данные текущего пользователя. |
| `minisocial/services/feed.py` | `fetch_posts`, `toggle_post_like`, `attach_post_galleries`, `count_master_accounts`. |
| `minisocial/routes/*.py` | Вместо Blueprint — функции `register_routes(app)`. |
| `templates/` | `base.html`, `index.html`, `login.html`, `register.html`, `admin.html`. |
| `static/` | `style.css`, `app.js`, `pixel-editor.js`, `emoji-wc.js`. |
| `seed_demo.py` | Опциональные тестовые пользователи/посты/лайки; флаги окружения и `app_settings`. |

---

## 4. Конфигурация и окружение

Переменные читаются в `minisocial/config.py` (и в `seed_demo.py` для демо).

| Переменная | Смысл |
|------------|--------|
| `FLASK_ENV` | Если `production`, `IS_PRODUCTION` истинно: обязательны `FLASK_SECRET_KEY` и учётные данные master-админа. |
| `FLASK_SECRET_KEY` | Секрет подписи сессий Flask. В dev при отсутствии — фиксированная строка (только не production). |
| `DATABASE_PATH` | Путь к файлу SQLite (по умолчанию `minisocial.db` в корне проекта). |
| `REGISTRATION_ENABLED_DEFAULT` | Значение по умолчанию для новых БД, если в `app_settings` нет `registration_enabled` (`true`/`1`/…). |
| `MASTER_ADMIN_USERNAME` / `MASTER_ADMIN_PASSWORD` | Если **нет** пользователя с `role = 'master'`, создаётся один. В production оба должны быть заданы. **В dev при отсутствии:** логин `admin`, пароль `admin12345`. |
| `MINISOCIAL_SKIP_AUTO_DEMO_SEED` | При `1`/`true`/… не вызывать `seed_demo.run_demo_seed(auto=True)` при старте (используется CLI `python seed_demo.py`). |

**Демо-заполнение (`seed_demo.py`):**

| Переменная | Смысл |
|------------|--------|
| `DEMO_SEED_ENABLED` | Должно быть истинно для автоматического или ручного запуска сида. |
| `DEMO_SEED_USERS` | Число пользователей `demo_seed_XX` (минимум 2 для взаимных лайков). |
| `DEMO_SEED_FORCE` | Только CLI: удаляет прежних `demo_seed_*` и пересоздаёт; сбрасывает ключ настройки `demo_seed_v1`. |

---

## 5. Константы (правила)

Заданы в `minisocial/config.py`, если не указано иное.

| Константа | Значение | Применение |
|-----------|----------|------------|
| `USERNAME_PATTERN` | `^[a-zA-Z0-9_]{3,32}$` | Регистрация и создание пользователя из админки. |
| `MAX_POST_CONTENT_LENGTH` | 280 | Текст поста. |
| `MAX_POST_IMAGE_BYTES` | 300 КиБ | На одно загруженное изображение в галерее. |
| `MAX_IMAGES_PER_POST` | 4 | Слотов в галерее; URL и файлы делят этот лимит. |
| `MAX_IMAGE_URL_LENGTH` | 1024 | Внешний URL картинки. |
| `ALLOWED_IMAGE_MIME_TYPES` | jpeg, png, webp, gif | Загрузки и отдача blob в постах. |
| `PIXEL_ART_SIZE` | 8 | Аватар строго PNG 8×8 (проверка IHDR). |
| `MAX_AVATAR_BYTES` | 64 КиБ | Загрузка аватара. |
| `AVATAR_MIME` | `image/png` | MIME сохранённого аватара. |
| `TRENDING_LIKE_WEIGHT` | 3.0 | Вес лайков в SQL для тренда. |
| `TRENDING_DECAY_PER_HOUR` | 0.25 | Затухание по часам в формуле тренда. |

---

## 6. Схема БД и миграции

**Подключение:** `get_db_connection()` включает внешние ключи и доступ к строкам как `Row`.

### Таблицы (логически)

1. **`users`** — `id`, `username` (уникальный), `password_hash`, `role` (`user` | `admin` | `master`), `status` (`active` | `archived`), `created_at`, `updated_at`, `avatar_blob`, `avatar_mime`.

2. **`posts`** — `id`, `content`, `likes` (денормализованный счётчик), `created_at`, `updated_at`, плюс мигрированные поля: `author_id`, `author_name_snapshot`, `author_state` (`active` | `archived` | `deleted`), `content_backup`, устаревшие `image_blob`, `image_mime`, `image_url` (в новых постах в основной строке часто NULL; галерея в `post_images`).

3. **`post_likes`** — `id`, `post_id`, `user_id`, `created_at`, **UNIQUE(post_id, user_id)**.

4. **`app_settings`** — `key`, `value` (например `registration_enabled`, `demo_seed_v1`).

5. **`post_images`** — `id`, `post_id`, `position` (слот с нуля), `image_blob`, `image_mime`, `image_url` (в строке либо blob, либо URL), **UNIQUE(post_id, position)**, FK `post_id` → `posts` ON DELETE CASCADE.

**Поведение миграций:** `init_db()` создаёт таблицы, добавляет недостающие столбцы, переносит старые одиночные картинки в `post_images`, переименовывает роль `master_admin` → `master`, задаёт настройку регистрации по умолчанию, гарантирует наличие **одного** пользователя **master** (из env или дефолтов), синхронизирует `posts.likes` с подсчётом по `post_likes`.

**Проверка PNG:** `validate_png_dimensions(data, width, height)` читает IHDR без Pillow, чтобы зафиксировать размер аватара.

---

## 7. Роли и права

| Роль | Лента | Пост / лайк | Удалить свой пост | Админка | Вкл/выкл регистрации | Создать user/admin | Архив | Восстановить | Удалить навсегда |
|------|-------|-------------|-------------------|---------|----------------------|---------------------|--------|--------------|------------------|
| **user** | да | да | да | нет | нет | нет | нет | нет | нет |
| **admin** | да | да | да | да | нет | нет | только роль **user** | только **user** | нет |
| **master** | да | да | да | да | да | да (`user` или `admin`) | по правилам ниже | по правилам ниже | да (не master, не себя) |

**Кто может удалить пост:** автор **или** `admin` **или** `master` (`current_user_can_manage_post`).

**Архивация:** нельзя архивировать себя; нельзя архивировать `master`; админ не архивирует админа; master может архивировать `admin` и `user`. Последнего master нельзя архивировать (проверка `count_master_accounts`).

**Восстановление / удаление:** те же ограничения по ролям; для аккаунтов admin — только master; окончательное удаление — только master.

**Вход архивированного:** сессия сбрасывается; сообщение об архивном аккаунте.

---

## 8. HTTP-маршруты (полный список)

### Лента

| Метод | Путь | Авторизация | Поведение |
|-------|------|-------------|-----------|
| GET | `/` | — | Редирект на `feed_newest`. |
| GET | `/feed/newest` | — | `fetch_posts("newest")` → `index.html`, `feed_type=newest`. |
| GET | `/feed/trending` | — | `fetch_posts("trending")` → `index.html`, `feed_type=trending`. |

### Аккаунты

| Метод | Путь | Авторизация | Поведение |
|-------|------|-------------|-----------|
| GET/POST | `/register` | — | GET: форма, если `is_registration_enabled()`; POST: при выключенной регистрации — flash и редирект на login; валидация логина/пароля; хеш; роль `user`, статус `active`. |
| GET/POST | `/login` | — | POST: проверка хеша; отказ для archived; `session`: `user_id`, `username`, `role`; редирект на ленту. |
| POST | `/logout` | `login_required` | Очистка сессии; редирект на login. |

### Посты и лайки

| Метод | Путь | Авторизация | Поведение |
|-------|------|-------------|-----------|
| POST | `/create-post` | `login_required` | Длина текста; `gallery_count` и `kind` слота (`url` \| `file`); URL: `http`/`https`, лимит длины; файлы: MIME из списка, лимит размера; INSERT в `posts` и строки `post_images`; flash об успехе. |
| POST | `/delete-post/<id>` | `login_required` | Загрузка поста; только автор или admin/master; DELETE поста (каскад по схеме). |
| POST | `/like-post/<id>` | `login_required` | Переключение лайка; flash; редирект на ленту (если JS выключен). |
| POST | `/api/posts/<id>/like` | `login_required` | JSON: `{ ok, likes, liked }` или 404. |

### Медиа

| Метод | Путь | Авторизация | Поведение |
|-------|------|-------------|-----------|
| GET | `/user-avatar/<user_id>` | — | Нет пользователя / нет аватара / статус ≠ `active` / MIME ≠ `image/png` → **404**. Иначе байты PNG, `Cache-Control: private, no-cache`. |
| POST | `/profile/avatar` | `login_required` | Один файл: только PNG, ≤64 КиБ, IHDR 8×8; UPDATE `users` для blob/mime. |
| GET | `/post-image/<post_id>/<slot>` | — | Слот 0…`MAX_IMAGES_PER_POST-1`. Отдача blob только если есть blob, MIME разрешён, **`posts.author_state == 'active'`**; иначе 404. Кэш `public, max-age=3600`. |

### Админка

| Метод | Путь | Декоратор | Поведение |
|-------|------|-----------|-----------|
| GET | `/admin` | `admin_or_master_required` | Список пользователей (порядок: master, admin, затем user по имени); флаг регистрации; `can_manage_master_controls`, если роль master. |
| POST | `/admin/toggle-registration` | `master_required` | `action` `on` или `off` → `app_settings.registration_enabled`. |
| POST | `/admin/create-user` | `master_required` | Правила логина/пароля; роль только `user` или `admin`. |
| POST | `/admin/users/<id>/archive` | `admin_or_master_required` | По правилам выше; пользователь `archived`; посты: при необходимости бэкап в `content_backup`, `content` = placeholder `post from deleted user`, `author_state='archived'`. |
| POST | `/admin/users/<id>/restore` | `admin_or_master_required` | Пользователь `active`; восстановление `content` из `content_backup`, сброс бэкапа, `author_state='active'` для архивных постов. |
| POST | `/admin/users/<id>/delete` | `admin_or_master_required` | **Только master**; нельзя удалить себя/master/последнего master; посты анонимизируются: `author_id` NULL, `author_name_snapshot` `'user deleted'`, placeholder в `content`, `author_state='deleted'`; DELETE пользователя. |

---

## 9. Логика выборки ленты (`services/feed.py`)

**Базовый JOIN:** `posts` LEFT JOIN `users` по автору; LEFT JOIN `post_likes` как `user_likes` для id **текущего** пользователя — в каждой строке `liked_by_current_user` (0/1). Флаг `author_has_avatar` из `users.avatar_blob`.

**Newest:** ORDER BY `created_at DESC`, `id DESC`; во внешнем SELECT колонка `trending_score` NULL.

**Trending:** базовый запрос как подзапрос `feed_data`; считается:

\[
\text{trending\_score} = \text{ROUND}\bigl(\text{likes} \times \text{TRENDING\_LIKE\_WEIGHT} - (\text{julianday('now')} - \text{julianday(created\_at)}) \times 24 \times \text{TRENDING\_DECAY\_PER\_HOUR},\ 2\bigr)
\]

ORDER BY `trending_score DESC`, `created_at DESC`, `id DESC`.

**Галереи:** после выборки `attach_post_galleries` подгружает `post_images` для этих id постов (в списке нет blob) — у каждого элемента: `position`, `has_blob`, `image_url`, `image_mime`.

---

## 10. Лайки (`toggle_post_like`)

- Если в `post_likes` уже есть (post, user): удалить строку, уменьшить `posts.likes` (не ниже 0); вернуть `(liked=False, new_count)`.
- Иначе: INSERT, увеличить `posts.likes`; вернуть `(liked=True, new_count)`.
- В `init_db` также `sync_post_like_counts` для согласования денормализации при существующих строках в `post_likes`.

---

## 11. Поведение UI (шаблоны + JS)

### Общая вёрстка (`base.html`)

- **Боковая панель:** вход/регистрация или блок «вошли» с плейсхолдером аватара или картинкой с `/user-avatar/<id>`, ссылка на ленту, переключатель темы, ссылка Admin при `admin`/`master`, выход POST.
- **Тема:** ключ `localStorage` `minisocial-theme` (`light` \| `dark`); инлайн-скрипт до отрисовки задаёт класс `html.dark-mode`; переключатель обновляет `aria-checked`.
- **Flash-сообщения** в колонке ленты.
- **Лайтбокс:** полноэкранный просмотр превью галереи; назад/вперёд; Escape; клик по затемнению закрывает (`app.js`).
- **Модальное окно пиксель-редактора** (только для авторизованных): холст 8×8 с масштабом 32×, карандаш/ластик/очистка, выбор цвета, Save PNG.

### Страница ленты (`index.html`)

- Вкладки: **Newest** / **Trending** (активный класс).
- Подсказка гостю: нужен вход для поста и лайков.
- Каждый пост: строка автора — аватар при активном авторе с аватаром; правила имени для `deleted` / `archived` / обычный / unknown; текст; сетка **gallery** (в CSS-классе число); мета с временем и опционально **trending score**; кнопка лайка (или только число); удаление, если разрешено.
- **FAB** открывает модалку создания поста (авторизованные).
- **Модалка поста:** textarea (maxlength с сервера), счётчик символов, сборка галереи (файл, URL, Draw 8×8), кнопка эмодзи, отправка на `create-post` с динамически добавленными скрытыми полями и file inputs (`app.js`).

### Админка (`admin.html`)

- Только master: переключатель регистрации (форма с скрытым `action` on/off), форма создания пользователя.
- Таблица пользователей: Archive / Restore / Delete по правилам.

### Клиентские скрипты

- **`app.js`:** тема; открытие/закрытие модалки поста; список галереи и проверки; динамические поля; счётчик; авто-высота textarea; лайтбокс; `confirm()` при удалении поста/пользователя; **формы лайков** перехватывают submit → `fetch` POST `/api/posts/<id>/like` → обновление без перезагрузки; отправка формы переключения регистрации.
- **`pixel-editor.js`:** `openPixelEditor({ target: 'avatar' | 'compose' })` — для аватара подгружается текущая картинка в сетку; для поста отдельный буфер; экспорт PNG через `canvas.toBlob`; аватар отправляет скрытую `avatar-upload-form`; пост вызывает `addComposeGalleryItem(file)`.
- **`emoji-wc.js`:** загрузка `emoji-picker-element` с CDN; вставка unicode в позицию курсора с учётом `maxLength`; тема пикера синхронизирована с `html.dark-mode`; `window.closeEmojiWcPopover`.

---

## 12. Демо-заполнение (`seed_demo.py`)

При `DEMO_SEED_ENABLED` = true:

- `BEGIN IMMEDIATE` для блокировок SQLite при старте нескольких воркеров.
- Пропуск, если `app_settings.demo_seed_v1` = `1`, кроме CLI с `DEMO_SEED_FORCE`.
- Пользователи `demo_seed_00` … с общим хешированным паролем **`DemoSeed123!`**, случайные аватары PNG 8×8.
- У каждого 2–5 постов со случайным текстом из `SNIPPETS`; 0–4 изображений на пост (случайные PNG; при превышении лимита — запасной маленький PNG).
- Случайные лайки между разными пользователями; в конце `sync_post_like_counts`.
- Запись `demo_seed_v1`, чтобы не запускать повторно.

---

## 13. Безопасность и краевые случаи (как в эталоне)

- **Сессии:** обычная cookie-сессия Flask (в production задать `SECRET_KEY`).
- **CSRF:** явных токенов в формах нет (упрощение для курса); в продакшене можно добавить.
- **Пароли:** только хеши Werkzeug.
- **Перечисление пользователей:** сообщения входа/регистрации по возможности нейтральные.
- **Внешние картинки в ленте:** у `<img>` указано `referrerpolicy="no-referrer"`.
- **Архивированные авторы:** отдача blob по маршруту поста требует `author_state == 'active'` — загрузки архивных пользователей не отдаются.
- **Посты удалённого пользователя:** отображение `user deleted` и анонимизированный контент по правилам шаблонов.

---

## 14. Чеклист точного воссоздания

- [ ] Фабрика Flask с корректными путями к шаблонам/статике и `init_db` при старте.
- [ ] Схема SQLite + миграции + создание master + синхронизация счётчиков лайков.
- [ ] Переключатель регистрации в БД + значение по умолчанию из env.
- [ ] Login / register / logout с проверкой роли и статуса.
- [ ] Ленты newest и trending с SQL-формулой и метаданными галерей.
- [ ] Создание поста с multipart-галереей (слоты файл + URL) и проверкой на сервере.
- [ ] Переключение лайка с UNIQUE и денормализованным счётчиком.
- [ ] JSON API лайка + запасная форма без JS.
- [ ] Загрузка аватара: только PNG 8×8; отдача с проверкой активного пользователя.
- [ ] Отдача картинок поста: allowlist MIME и проверка активного автора.
- [ ] Админка: матрица ролей для архива/восстановления/удаления, регистрация, создание пользователя.
- [ ] Фронт: тёмная тема, модалка поста, галерея, эмодзи-пикер, пиксель-редактор (аватар + пост), лайтбокс, подтверждения, опциональный демо-сид.

Используйте документ как единый источник **поведенческих требований**; имена файлов и функций в эталонном репозитории напрямую указывают на детали реализации.
