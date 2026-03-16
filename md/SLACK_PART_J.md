{% raw %}
# Часть J: Ответы в тредах — создание и удаление

В Части I вы ввели таблицу `messages`, список тредов и просмотр выбранного треда (корневое сообщение и ответы), создание корневого сообщения (треда) и его мягкое удаление. Ответы в этой части только отображались; создавать и удалять их было нельзя.

В этой части вы добавляете **ответы (replies)**:

- **форма ответа** под выбранным тредом: одно поле и кнопка «Ответить»; та же проверка прав, что и для создания треда (писать могут только участник, владелец канала или админ; при роли «только чтение» форма не показывается, остаётся пояснение);
- **удаление ответа**: те же правила, что и для корневого сообщения (автор, владелец канала или админ); мягкое удаление, в интерфейсе — «[Удалено]», ответ остаётся на месте в списке ответов;
- по желанию: **подтверждение** перед удалением ответа; единообразное отображение удалённых тредов в списке (в Части I они уже показываются как «[Удалено]»).

Отдельная таблица для ответов не нужна: ответ — это сообщение с заполненным `parent_id`. Обработчик удаления из Части I (`message_delete_view`) уже применим к любому сообщению, поэтому для ответов достаточно добавить в шаблоне кнопку удаления и при необходимости подтверждение.

---

## J1. Создание ответа (reply)

Ответ — это сообщение в том же канале с `parent_id`, равным id корневого сообщения треда. Пользователь вводит текст в форме под тредом и отправляет POST; сервер проверяет право писать и что родительское сообщение действительно корневое в этом канале, затем вставляет строку в `messages`.

### Шаг J1.1. Обработчик создания ответа

Добавьте в `views_channels.py` функцию, обрабатывающую POST создания ответа. Маршрут может быть вида `POST /channels/<channel_id>/threads/<parent_id>/reply`: в URL передаётся id канала и id корневого сообщения (родителя).

Логика:

1. Проверка входа, загрузка канала (404 при отсутствии).
2. Проверка доступа к каналу: для приватного — только участник или админ (как в `channel_view_view`).
3. Проверка права писать: роль в канале `member` или `owner`, либо `is_admin()`; при роли `read_only` и не админ — 403.
4. Загрузка родительского сообщения: оно должно существовать, принадлежать этому каналу и иметь `parent_id IS NULL` (корневое). Иначе 404.
5. Взять `content` из формы (обрезка пробелов); при пустом — flash и редирект на страницу канала с `?thread=parent_id`.
6. Вставить в `messages` строку: `channel_id`, `author_id`, `parent_id` = id корневого сообщения, `content`. Редирект на страницу канала с `?thread=parent_id`, чтобы остаться в просмотре того же треда.

Пример:

```python
def reply_create_view(channel_id: int, parent_id: int):
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
        return redirect(url_for("channel_view", channel_id=channel_id, thread=parent_id))
    if membership and membership["role"] == "read_only" and not is_admin():
        conn.close()
        abort(403)

    # Родительское сообщение должно быть корневым в этом канале
    parent = conn.execute(
        "SELECT id FROM messages WHERE id = ? AND channel_id = ? AND parent_id IS NULL",
        (parent_id, channel_id),
    ).fetchone()
    if not parent:
        conn.close()
        abort(404)

    content = (request.form.get("content") or "").strip()
    if not content:
        conn.close()
        flash("Введите текст ответа.")
        return redirect(url_for("channel_view", channel_id=channel_id, thread=parent_id))

    conn.execute(
        "INSERT INTO messages (channel_id, author_id, parent_id, content) VALUES (?, ?, ?, ?)",
        (channel_id, user["id"], parent_id, content),
    )
    conn.commit()
    conn.close()

    flash("Ответ добавлен.")
    return redirect(url_for("channel_view", channel_id=channel_id, thread=parent_id))
```

### Шаг J1.2. Маршрут и форма ответа в шаблоне

В `app.py` добавьте импорт `reply_create_view` и маршрут:

```python
from views_channels import (
    ...
    thread_create_view,
    message_delete_view,
    reply_create_view,
)
```

```python
@app.post("/channels/<int:channel_id>/threads/<int:parent_id>/reply")
def reply_create(channel_id, parent_id):
    return reply_create_view(channel_id, parent_id)
```

В `templates/channel_view.html` внутри блока `{% if selected_thread %}` после заголовка «Ответы» и списка ответов добавьте форму ответа **только для пользователей с правом писать** (`can_post`). Форма отправляет POST на `url_for('reply_create', channel_id=channel.id, parent_id=selected_thread.id)` с полем `name="content"`. Например, форма может стоять после текста «Нет ответов» / списка ответов и перед закрытием секции треда:

```html
        <h4>Ответы</h4>
        {% if not thread_replies %}
          <p class="muted">Нет ответов.</p>
        {% else %}
          <ul>
            {% for r in thread_replies %}
              <li>
                ...
              </li>
            {% endfor %}
          </ul>
        {% endif %}

        {% if can_post %}
          <form method="post" action="{{ url_for('reply_create', channel_id=channel.id, parent_id=selected_thread.id) }}" class="form" style="margin-top: 12px;">
            <label for="reply_content">Ответить</label>
            <input type="text" id="reply_content" name="content" placeholder="Текст ответа" required />
            <button type="submit" class="btn btn-primary">Ответить</button>
          </form>
        {% endif %}
      </section>
    {% endif %}
```

Таким образом, форма ответа видна только при открытом треде и только тем, кто может писать в канале (участник, владелец канала, админ). Пользователи с ролью «только чтение» по-прежнему видят только пояснение в блоке «Сообщения» и не видят форму ответа.

---

## J2. Удаление ответа

Удаление ответа использует тот же механизм, что и в Части I: один и тот же обработчик `message_delete_view` и маршрут `POST /channels/<id>/messages/<message_id>/delete`. Удалять могут автор сообщения, владелец канала или админ; выполняется мягкое удаление (`deleted_at`, очистка `content`). В интерфейсе ответ остаётся на своей позиции в списке ответов, вместо текста показывается «[Удалено]» (это уже реализовано в Части I для `thread_replies`).

### Шаг J2.1. Кнопка «Удалить» у каждого ответа

В шаблоне для каждого элемента в `thread_replies` нужно показывать кнопку «Удалить» только если текущий пользователь имеет право удалить это сообщение: он автор ответа, владелец канала или админ. В шаблоне уже доступны: `current_user()` и `is_admin()` (из context_processor), `channel` (в том числе `channel.owner_id`), у каждого ответа `r` есть `r.author_id`. Значит, условие «можно удалить этот ответ» можно записать так: автор ответа — текущий пользователь, или владелец канала — текущий пользователь, или глобальный админ. Учтите, что `channel.owner_id` может быть `None` (канал без владельца); тогда сравнение с `current_user().id` даёт False, и удалять сможет только автор или админ.

Добавьте рядом с каждым ответом форму с кнопкой «Удалить», отправляющую POST на `url_for('message_delete', channel_id=channel.id, message_id=r.id)`, обёрнутую в условие по праву удаления. По желанию добавьте подтверждение: `onsubmit="return confirm('Удалить этот ответ?');"` у формы.

Пример фрагмента списка ответов:

```html
        {% else %}
          <ul>
            {% for r in thread_replies %}
              <li>
                {% if r.deleted_at %}<span class="muted">[Удалено]</span>{% else %}{{ r.content }}{% endif %}
                <span class="muted"> — {{ r.author_name or '(автор удалён)' }}, {{ r.created_at }}</span>
                {% if not r.deleted_at and (r.author_id == current_user().id or channel.owner_id == current_user().id or is_admin()) %}
                  <form method="post" action="{{ url_for('message_delete', channel_id=channel.id, message_id=r.id) }}" style="display: inline;" onsubmit="return confirm('Удалить этот ответ?');">
                    <button type="submit" class="btn btn-small">Удалить</button>
                  </form>
                {% endif %}
              </li>
            {% endfor %}
          </ul>
        {% endif %}
```

Удалённые ответы (`r.deleted_at` задано) по-прежнему выводятся как «[Удалено]» без кнопки удаления. Позиция ответа в списке не меняется — порядок задаётся выборкой по `created_at` в представлении.

---

## J3. Опциональные доработки

### Список тредов: удалённые корневые сообщения

В Части I удалённые треды уже отображаются в списке как «[Удалено]» (с сохранением порядка и возможности открыть тред, чтобы видеть ответы). Если вы хотите **скрывать** удалённые корневые сообщения из списка тредов, в `channel_view_view()` измените выборку `thread_starters`: добавьте условие `AND m.deleted_at IS NULL`. Тогда такие треды не будут видны в списке, но записи в БД останутся (мягкое удаление). Альтернатива — оставить как есть и единообразно показывать «[Удалено]» везде (список и просмотр треда).

### Подтверждение удаления ответа

В шаге J2.1 уже приведён вариант с `onsubmit="return confirm('Удалить этот ответ?');"`. При желании текст можно изменить или убрать подтверждение, оставив только форму и кнопку.

---

## Итог Части J (Slack)

После выполнения этой части в проекте есть:

- **Создание ответа**: форма «Ответить» под выбранным тредом (поле `content`), доступная при `can_post`. Обработчик `reply_create_view(channel_id, parent_id)`: проверка доступа к каналу и права писать, проверка что `parent_id` — id корневого сообщения в этом канале, вставка в `messages` с заданным `parent_id`. Маршрут `POST /channels/<id>/threads/<parent_id>/reply`.
- **Удаление ответа**: тот же `message_delete_view` и маршрут `POST /channels/<id>/messages/<message_id>/delete`. В шаблоне у каждого ответа при наличии права (автор, владелец канала или админ) выводится кнопка «Удалить» с опциональным подтверждением. Удалённые ответы отображаются как «[Удалено]» и остаются на месте в списке ответов.

Права на создание и удаление ответов совпадают с правами для тредов: писать и удалять могут участник (member), владелец канала (owner) или админ; роль «только чтение» не видит форму ответа. На этом базовая функциональность сообщений и тредов (корневые сообщения и ответы, создание и мягкое удаление) завершена.
{% endraw %}
