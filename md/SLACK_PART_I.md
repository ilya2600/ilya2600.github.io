{% raw %}
# Часть I: Сообщения и треды — модель, отображение, создание и удаление тредов

В Части H вы добавили управление участниками канала: блок «Участники» для владельца и админа, приглашение и удаление участников. Блок «Сообщения» на странице канала пока остаётся заглушкой.

В этой части вы вводите **модель сообщений и тредов**:

- таблицу `messages` в `db.py` (сообщение привязано к каналу и автору; у ответов задаётся родительское сообщение);
- на странице канала — **список тредов** (корневых сообщений) и **просмотр выбранного треда** (корень и ответы в хронологическом порядке);
- **создание нового треда** (корневое сообщение): форма и проверка прав (писать могут только участник, владелец канала или админ; роль «только чтение» видит пояснение без формы);
- **удаление корневого сообщения** (треда): мягкое удаление (`deleted_at`), права (автор, владелец канала, админ), отображение «[Удалено]» в списке и в просмотре, подтверждение перед удалением.

Ответы на треды (reply) в этой части не создаются — только модель и отображение уже существующих ответов. Создание и удаление ответов будет в следующей части.

---

## I1. Таблица `messages` в `db.py`

Сообщение принадлежит каналу и пользователю-автору. Оно может быть **корневым** (старт треда) или **ответом**: у ответа задаётся `parent_id` — id корневого сообщения. Для корневых сообщений `parent_id` храним как `NULL`. Мягкое удаление: поле `deleted_at`; при удалении выставляем дату и по желанию очищаем `content` или скрываем его в интерфейсе.

### Шаг I1.1. Поля таблицы `messages`

Таблица хранит: `id`, `channel_id`, `author_id`, `parent_id` (NULL для корневых), `content` (текст сообщения), `created_at`, `deleted_at` (NULL, если сообщение не удалено). Ссылки на `channels(id)` и `users(id)`; для `parent_id` — на `messages(id)`.

### Шаг I1.2. Добавляем таблицу в `init_db()`

Откройте `db.py` и найдите функцию `init_db()`. Сразу **после** блока `CREATE TABLE IF NOT EXISTS channel_members (...)` добавьте создание таблицы `messages`:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS messages (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            channel_id INTEGER NOT NULL,
            author_id INTEGER,
            parent_id INTEGER,
            content TEXT NOT NULL DEFAULT '',
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            deleted_at TEXT,
            FOREIGN KEY (channel_id) REFERENCES channels(id),
            FOREIGN KEY (author_id) REFERENCES users(id),
            FOREIGN KEY (parent_id) REFERENCES messages(id)
        )
    """)
```

После удаления автора или канала строки в `messages` можно оставить (author_id/channel_id не обнулять), либо доработать логику позже. Для учебного проекта достаточно такой схемы.

В итоге `init_db()` создаёт таблицы `settings`, `users`, `channels`, `channel_members`, `messages`, затем выполняет `conn.commit()` и `conn.close()`.

---

## I2. Страница канала: список тредов и просмотр одного треда

На странице канала вместо заглушки «Сообщения» нужно показать список **корневых сообщений** (тредов) и при выборе — сам тред: корневое сообщение и все ответы по времени. Выбор треда удобно сделать через query-параметр, например `?thread=<id>`: тот же маршрут `GET /channels/<id>` принимает опциональный `thread` и отдаёт страницу с уже выбранным тредом.

### Шаг I2.1. Выборка списка тредов и выбранного треда в `channel_view_view()`

В `views_channels.py` в функции `channel_view_view()` после всех текущих выборок (участники, invite_candidates) и **до** `conn.close()` добавьте:

1. Список корневых сообщений канала (тредов): `parent_id IS NULL`, по `created_at`. В списке показываем и удалённые (чтобы тред с ответами оставался доступным); в шаблоне выводим текст или «[Удалено]» по наличию `deleted_at`.
2. Опционально выбранный тред: из `request.args.get('thread')` получаем id сообщения; если передан и сообщение существует, принадлежит каналу и является корневым (`parent_id IS NULL`), загружаем его и все ответы (сообщения с `parent_id = thread_id`) по `created_at`.

Переменные для шаблона: `thread_starters` (список корневых сообщений с полями, достаточными для списка: id, content, author_id/username, created_at, deleted_at) и при выборе — `selected_thread` (корневое сообщение) и `thread_replies` (ответы). Для отображения имени автора в списке и в просмотре нужен join с `users` по `author_id`.

Пример запросов (после блока с `invite_candidates`, перед `conn.close()`):

```python
    # Список тредов (корневых сообщений) канала — для блока «Сообщения»
    thread_starters = conn.execute("""
        SELECT m.id, m.content, m.author_id, m.created_at, m.deleted_at,
               u.username AS author_name
        FROM messages m
        LEFT JOIN users u ON m.author_id = u.id
        WHERE m.channel_id = ? AND m.parent_id IS NULL
        ORDER BY m.created_at DESC
    """, (channel_id,)).fetchall()

    # Выбранный тред (из query-параметра ?thread=id)
    selected_thread = None
    thread_replies = []
    thread_id_param = request.args.get("thread")
    if thread_id_param:
        try:
            tid = int(thread_id_param)
            row = conn.execute("""
                SELECT m.id, m.content, m.author_id, m.created_at, m.deleted_at,
                       u.username AS author_name
                FROM messages m
                LEFT JOIN users u ON m.author_id = u.id
                WHERE m.id = ? AND m.channel_id = ? AND m.parent_id IS NULL
            """, (tid, channel_id)).fetchone()
            if row:
                selected_thread = row
                thread_replies = conn.execute("""
                    SELECT m.id, m.content, m.author_id, m.created_at, m.deleted_at,
                           u.username AS author_name
                    FROM messages m
                    LEFT JOIN users u ON m.author_id = u.id
                    WHERE m.parent_id = ?
                    ORDER BY m.created_at ASC
                """, (tid,)).fetchall()
        except (TypeError, ValueError):
            pass
```

Добавьте в `render_template()` переменные: `thread_starters`, `selected_thread`, `thread_replies`.

### Шаг I2.2. Флаг «может писать» в канале

Для формы создания треда и для отображения пояснения «только чтение» нужен флаг: текущий пользователь может писать, если его роль в канале `member` или `owner`, либо он глобальный админ. Роль `read_only` не может писать.

Добавьте перед `conn.close()` (например, после выборки `thread_replies`):

```python
    can_post = my_role in ("member", "owner") or is_admin()
```

Передайте `can_post` в шаблон. Если пользователь не в канале (`my_role` нет), считаем, что писать нельзя (`can_post = False` уже при `my_role is None`; при необходимости задайте явно: `can_post = (my_role in ("member", "owner") or is_admin()) if my_role else False`).

### Шаг I2.3. Шаблон: список тредов и просмотр треда

В `templates/channel_view.html` замените блок «Сообщения» (заголовок и заглушка) на:

1. Заголовок «Сообщения» / «Треды».
2. Если есть право писать (`can_post`) — форма «Новый тред» (одно поле, например `content` или `title`, и кнопка). Обработчик формы добавим в разделе I3.
3. Список тредов: ссылки вида «первые слова / дата» или «[Удалено]» + дата, ведущие на тот же URL с `?thread=<id>`. Для отображения текста: если у сообщения есть `deleted_at`, выводить «[Удалено]», иначе — фрагмент `content` или весь текст.
4. Если выбран тред (`selected_thread` передан): блок с заголовком «Тред» и корневым сообщением (автор, дата, текст или «[Удалено]»), затем список `thread_replies` (автор, дата, текст или «[Удалено]»). Ответов в этой части ещё нет, но структура готова.

Пример структуры (форма создания — в I3):

```html
    <h2>Сообщения</h2>

    {% if can_post %}
      <form method="post" action="{{ url_for('thread_create', channel_id=channel.id) }}" class="form" style="margin-bottom: 16px;">
        <label for="content">Новый тред</label>
        <input type="text" id="content" name="content" placeholder="Тема или первое сообщение" required />
        <button type="submit" class="btn btn-primary">Создать тред</button>
      </form>
    {% else %}
      {% if my_role == 'read_only' %}
        <p class="muted">В этом канале вы можете только просматривать сообщения.</p>
      {% endif %}
    {% endif %}

    <h3>Треды</h3>
    {% if not thread_starters %}
      <p class="muted">Пока нет тредов.</p>
    {% else %}
      <ul>
        {% for t in thread_starters %}
          <li>
            <a href="{{ url_for('channel_view', channel_id=channel.id, thread=t.id) }}">{{ t.content[:50] if not t.deleted_at else '[Удалено]' }}{% if t.content|length > 50 and not t.deleted_at %}…{% endif %}</a>
            <span class="muted"> — {{ t.author_name or '(автор удалён)' }}, {{ t.created_at }}</span>
          </li>
        {% endfor %}
      </ul>
    {% endif %}

    {% if selected_thread %}
      <section class="card" style="margin-top: 16px;">
        <h3>Тред</h3>
        <p>
          {% if selected_thread.deleted_at %}<span class="muted">[Удалено]</span>{% else %}{{ selected_thread.content }}{% endif %}
          <span class="muted"> — {{ selected_thread.author_name or '(автор удалён)' }}, {{ selected_thread.created_at }}</span>
          {% if can_delete_thread %}
            <form method="post" action="{{ url_for('message_delete', channel_id=channel.id, message_id=selected_thread.id) }}" style="display: inline;" onsubmit="return confirm('Удалить этот тред?');">
              <button type="submit" class="btn btn-small">Удалить тред</button>
            </form>
          {% endif %}
        </p>
        <h4>Ответы</h4>
        {% if not thread_replies %}
          <p class="muted">Нет ответов. (Добавление ответов — в следующей части.)</p>
        {% else %}
          <ul>
            {% for r in thread_replies %}
              <li>
                {% if r.deleted_at %}<span class="muted">[Удалено]</span>{% else %}{{ r.content }}{% endif %}
                <span class="muted"> — {{ r.author_name or '(автор удалён)' }}, {{ r.created_at }}</span>
              </li>
            {% endfor %}
          </ul>
        {% endif %}
      </section>
    {% endif %}
```

Здесь использованы маршруты `thread_create` и `message_delete` и флаг `can_delete_thread` — они появятся в шагах I3 и I4. В шаблоне можно временно убрать кнопку удаления и форму нового треда, пока соответствующие представления не добавлены, либо оставить разметку и подключить маршруты дальше.

---

## I3. Создание треда (корневое сообщение)

Только пользователи с правом писать в канале могут создавать треды: роль в канале `member` или `owner`, либо глобальный админ. Участники с ролью `read_only` не видят форму и видят пояснение (как в примере выше).

### Шаг I3.1. Обработчик создания треда

Добавьте в `views_channels.py` функцию, обрабатывающую POST создания треда (например, маршрут `POST /channels/<id>/threads`). Поле формы — `content` (текст корневого сообщения).

Логика:

1. Проверка входа, загрузка канала (404 при отсутствии).
2. Проверка доступа к каналу: для приватного — только участник или админ (как в `channel_view_view`).
3. Проверка права писать: роль в канале `member` или `owner`, либо `is_admin()`; иначе 403 или редирект с сообщением.
4. Взять `content` из формы (обрезка пробелов); при пустом — сообщение и редирект на страницу канала.
5. Вставить в `messages` строку: `channel_id`, `author_id`, `parent_id = NULL`, `content`, `created_at` по умолчанию, `deleted_at = NULL`.
6. Редирект на страницу канала (можно с `?thread=<новый_id>`, чтобы сразу открыть созданный тред).

Пример:

```python
def thread_create_view(channel_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    conn = get_conn()

    ch = conn.execute(
        "SELECT id, name, type, owner_id FROM channels WHERE id = ?",
        (channel_id,),
    ).fetchone()

    if ch is None:
        conn.close()
        abort(404)

    # Доступ к каналу: приватный — только участник или админ
    if ch["type"] == "private" and not is_admin():
        member = conn.execute(
            "SELECT 1 FROM channel_members WHERE channel_id = ? AND user_id = ?",
            (channel_id, user["id"]),
        ).fetchone()
        if not member:
            conn.close()
            abort(403)

    # Писать могут member, owner или админ
    membership = conn.execute(
        "SELECT role FROM channel_members WHERE channel_id = ? AND user_id = ?",
        (channel_id, user["id"]),
    ).fetchone()
    if not membership and not is_admin():
        conn.close()
        flash("Вы не состоите в канале.")
        return redirect(url_for("channel_view", channel_id=channel_id))
    if membership and membership["role"] == "read_only" and not is_admin():
        conn.close()
        abort(403)

    content = (request.form.get("content") or "").strip()
    if not content:
        conn.close()
        flash("Введите текст сообщения.")
        return redirect(url_for("channel_view", channel_id=channel_id))

    cur = conn.execute(
        "INSERT INTO messages (channel_id, author_id, parent_id, content) VALUES (?, ?, NULL, ?)",
        (channel_id, user["id"], content),
    )
    new_id = cur.lastrowid
    conn.commit()
    conn.close()

    flash("Тред создан.")
    return redirect(url_for("channel_view", channel_id=channel_id, thread=new_id))
```

Обратите внимание: для редиректа с query-параметром в Flask используется `url_for("channel_view", channel_id=channel_id, thread=new_id)` — параметр `thread` попадёт в URL как `?thread=...`.

### Шаг I3.2. Маршрут создания треда

В `app.py` добавьте импорт `thread_create_view` и маршрут:

```python
@app.post("/channels/<int:channel_id>/threads")
def thread_create(channel_id):
    return thread_create_view(channel_id)
```

Убедитесь, что форма в шаблоне отправляет POST на `url_for('thread_create', channel_id=channel.id)` с полем `name="content"`.

---

## I4. Удаление треда (корневого сообщения)

Удаление — мягкое: выставляем `deleted_at` (например, текущее время в ISO-формате) и по желанию очищаем `content`. Удалять могут автор сообщения, владелец канала или глобальный админ. В интерфейсе: в списке тредов и в просмотре показывать «[Удалено]» вместо текста. Перед удалением корневого сообщения запрашивать подтверждение (например, `onsubmit="return confirm('...');"`).

### Шаг I4.1. Обработчик удаления сообщения

Добавьте в `views_channels.py` функцию для POST-удаления сообщения (маршрут вида `POST /channels/<id>/messages/<message_id>/delete`). В этой части вызывать её только для корневых сообщений (тредов); в следующей части тот же обработчик можно использовать для ответов.

Логика:

1. Проверка входа, загрузка канала и сообщения (сообщение должно принадлежать каналу).
2. Проверка прав: удалять могут автор сообщения, владелец канала (`ch.owner_id == current_user.id`) или админ.
3. Установить у сообщения `deleted_at = strftime('%Y-%m-%dT%H:%M:%SZ', 'now')` и обнулить `content` (или оставить content и скрывать в шаблоне — по вашему выбору).
4. Редирект на страницу канала (без параметра `thread` или с ним, если нужно остаться в просмотре треда).

Пример:

```python
from datetime import datetime

# В начале файла добавьте импорт datetime, если его ещё нет (для deleted_at).
```

Или без отдельного импорта, используя SQLite:

```python
def message_delete_view(channel_id: int, message_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    conn = get_conn()

    ch = conn.execute(
        "SELECT id, owner_id FROM channels WHERE id = ?",
        (channel_id,),
    ).fetchone()

    if ch is None:
        conn.close()
        abort(404)

    msg = conn.execute(
        "SELECT id, author_id, parent_id FROM messages WHERE id = ? AND channel_id = ?",
        (message_id, channel_id),
    ).fetchone()

    if msg is None:
        conn.close()
        abort(404)

    # Удалять могут автор, владелец канала или админ
    if msg["author_id"] != user["id"] and ch["owner_id"] != user["id"] and not is_admin():
        conn.close()
        abort(403)

    conn.execute(
        "UPDATE messages SET deleted_at = strftime('%Y-%m-%dT%H:%M:%SZ', 'now'), content = '' WHERE id = ?",
        (message_id,),
    )
    conn.commit()
    conn.close()

    flash("Сообщение удалено.")
    return redirect(url_for("channel_view", channel_id=channel_id))
```

Редирект без `thread` возвращает на общий вид канала; при желании можно передать `thread=message_id`, чтобы остаться в просмотре этого треда (тогда будет видно «[Удалено]»).

### Шаг I4.2. Маршрут удаления и кнопка в шаблоне

В `app.py` добавьте импорт `message_delete_view` и маршрут:

```python
@app.post("/channels/<int:channel_id>/messages/<int:message_id>/delete")
def message_delete(channel_id, message_id):
    return message_delete_view(channel_id, message_id)
```

В шаблоне для выбранного треда нужен флаг «может удалить это сообщение»: автор, владелец канала или админ. Удобно передавать из представления переменную `can_delete_thread` для выбранного треда: True, если текущий пользователь — автор `selected_thread`, или владелец канала, или админ. Добавьте в `channel_view_view()` после определения `selected_thread` и `thread_replies`:

```python
    can_delete_thread = False
    if selected_thread:
        is_author = (selected_thread["author_id"] == user["id"])
        is_channel_owner = (ch["owner_id"] == user["id"])
        can_delete_thread = is_author or is_channel_owner or is_admin()
```

Передайте `can_delete_thread` в шаблон и используйте в блоке с кнопкой «Удалить тред», обернув форму в `{% if can_delete_thread %}`. На кнопку удаления добавьте подтверждение: `onsubmit="return confirm('Удалить этот тред?');"` у формы.

---

## I5. Итоговая вставка в `channel_view_view()` и шаблон

Ниже — сводка того, что должно оказаться в `channel_view_view()` перед `conn.close()` и в `render_template`, чтобы всё из разделов I2–I4 работало вместе.

В `views_channels.py` перед `conn.close()`:

- Выборка `thread_starters` (корневые сообщения канала с автором).
- Выборка `selected_thread` и `thread_replies` по `request.args.get("thread")`.
- Вычисление `can_post` (роль member/owner или админ).
- Вычисление `can_delete_thread` для выбранного треда (автор, владелец канала или админ).

В `render_template()` добавьте аргументы: `thread_starters`, `selected_thread`, `thread_replies`, `can_post`, `can_delete_thread`.

В шаблоне блок «Сообщения» заменён на: форма нового треда (при `can_post`), пояснение для read_only (при отсутствии права писать), список тредов со ссылками `?thread=...`, при выбранном треде — блок с корневым сообщением (и кнопкой удаления с подтверждением при `can_delete_thread`) и списком ответов. Удалённые сообщения отображаются как «[Удалено]».

---

## Итог Части I (Slack)

После выполнения этой части в проекте есть:

- **`db.py`**: таблица `messages` с полями `id`, `channel_id`, `author_id`, `parent_id`, `content`, `created_at`, `deleted_at`.
- **Страница канала**: список тредов (корневых сообщений), выбор треда по `?thread=<id>`, отображение треда (корень + ответы в хронологическом порядке); удалённые сообщения отображаются как «[Удалено]».
- **Создание треда**: форма «Новый тред» (поле `content`), доступная только при праве писать (`member`, `owner` или админ); для роли `read_only` выводится пояснение без формы. Маршрут `POST /channels/<id>/threads`.
- **Удаление сообщения (треда)**: мягкое удаление (`deleted_at`, очистка `content`), право удаления у автора, владельца канала и админа. Кнопка «Удалить тред» с подтверждением. Маршрут `POST /channels/<id>/messages/<message_id>/delete`.

Ответы на треды в этой части не создаются и не удаляются — только модель и отображение. В следующей части добавляются форма ответа и удаление ответов.
{% endraw %}
