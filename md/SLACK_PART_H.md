{% raw %}
# Часть H: Управление участниками канала — список, приглашение и удаление

В Части G вы ввели модель членства: таблицу `channel_members`, вступление в публичный канал и выход из канала, а также ограничение доступа к приватным каналам. Участник может вступить в публичный канал сам или выйти по кнопке «Покинуть канал». Кто может **приглашать** в канал других пользователей и **удалять** участников, до сих пор не было реализовано.

В этой части вы добавляете **управление участниками внутри канала**:

- блок **«Участники»** на странице канала: список участников с ролями, видимый владельцу канала и админу;
- **приглашение** пользователя: выбор из списка активных (не архивированных) пользователей, добавление в `channel_members` с ролью по типу канала (`member` или `read_only`);
- **удаление участника**: владелец или админ может удалить участника; если удалённый был владельцем канала — применяется та же логика передачи владения или канала без владельца, что и при выходе в Части G.

Сообщения и треды остаются на следующие части.

---

## H1. Блок «Участники» на странице канала

Блок должен быть виден только тем, кто может управлять участниками: **владелец канала** (текущий пользователь в `channel_members` с ролью `owner` или `ch.owner_id == current_user.id`) или **глобальный админ** (`is_admin()`). Обычные участники и гости канала этот блок не видят.

### Шаг H1.1. Выборка списка участников и флаг «может управлять»

В `views_channels.py` в функции `channel_view_view()` после вычисления `my_role` и **до** `conn.close()` нужно:

1. Выбрать всех участников канала с именами и ролями.
2. Определить, может ли текущий пользователь управлять участниками: он владелец канала или админ.

Список участников удобно получить соединением `channel_members` с `users` (чтобы подставить `username`). Так как соединение с БД в этот момент ещё открыто, добавьте запросы перед `conn.close()`:

```python
    # Список участников канала (id, username, role) — для блока «Участники»
    members = conn.execute("""
        SELECT u.id AS user_id, u.username, cm.role
        FROM channel_members cm
        INNER JOIN users u ON u.id = cm.user_id
        WHERE cm.channel_id = ?
        ORDER BY cm.role = 'owner' DESC, u.username ASC
    """, (channel_id,)).fetchall()

    # Управление участниками доступно владельцу канала или админу
    can_manage_members = (ch["owner_id"] == user["id"]) or is_admin()
```

Затем передайте в шаблон две новые переменные: `members` и `can_manage_members`:

```python
    return render_template(
        "channel_view.html",
        channel=ch,
        my_role=my_role,
        members=members,
        can_manage_members=can_manage_members,
    )
```

Порядок сортировки: сначала владелец (`role = 'owner'`), затем остальные по имени. В шаблоне блок «Участники» выводится только при `can_manage_members`.

### Шаг H1.2. Шаблон: блок «Участники»

В `templates/channel_view.html` добавьте секцию **после** блока с ролью и кнопками «Вступить» / «Покинуть канал» и **перед** заголовком «Сообщения». Секция отображается только если `can_manage_members` истина:

```html
    {% if can_manage_members %}
      <h2>Участники</h2>
      <table>
        <thead>
          <tr>
            <th>Пользователь</th>
            <th>Роль</th>
            <th></th>
          </tr>
        </thead>
        <tbody>
          {% for m in members %}
            <tr>
              <td>{{ m.username }}</td>
              <td>{{ m.role }}</td>
              <td>
                <form method="post" action="{{ url_for('channel_remove_member', channel_id=channel.id, user_id=m.user_id) }}" style="display: inline;">
                  <button type="submit" class="btn btn-small">Удалить</button>
                </form>
              </td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    {% endif %}
```

Кнопка «Удалить» пока ведёт на маршрут, который мы добавим в разделе H3. Если хотите сначала только список без кнопки — можно временно убрать форму и добавить её после реализации удаления.

---

## H2. Приглашение пользователя в канал

Владелец или админ может добавить в канал любого **активного** пользователя (то есть с `archived_at IS NULL`). Пользователи, уже состоящие в канале, в списке для приглашения не показываются. Роль при добавлении: если тип канала `read_only` — в `channel_members` ставим роль `read_only`, иначе — `member`.

### Шаг H2.1. Список пользователей для приглашения

В `channel_view_view()` при отображении страницы канала владельцу/админу нужен список пользователей, которых **ещё нет** в канале и которые **не архивированы**. Его можно передать в шаблон и использовать в выпадающем списке формы приглашения.

Добавьте выборку **только если** `can_manage_members` (чтобы не делать лишний запрос остальным):

```python
    # Список участников канала (id, username, role) — для блока «Участники»
    members = conn.execute("""
        SELECT u.id AS user_id, u.username, cm.role
        FROM channel_members cm
        INNER JOIN users u ON u.id = cm.user_id
        WHERE cm.channel_id = ?
        ORDER BY cm.role = 'owner' DESC, u.username ASC
    """, (channel_id,)).fetchall()

    # Управление участниками доступно владельцу канала или админу
    can_manage_members = (ch["owner_id"] == user["id"]) or is_admin()

    # Для приглашения: активные пользователи, ещё не в канале (только если показываем блок управления)
    invite_candidates = []
    if can_manage_members:
        invite_candidates = conn.execute("""
            SELECT u.id, u.username
            FROM users u
            WHERE u.archived_at IS NULL
              AND NOT EXISTS (
                  SELECT 1 FROM channel_members cm
                  WHERE cm.channel_id = ? AND cm.user_id = u.id
              )
            ORDER BY u.username
        """, (channel_id,)).fetchall()
```

Передайте `invite_candidates` в шаблон:

```python
    return render_template(
        "channel_view.html",
        channel=ch,
        my_role=my_role,
        members=members,
        can_manage_members=can_manage_members,
        invite_candidates=invite_candidates,
    )
```

### Шаг H2.2. Обработка приглашения (POST)

Добавьте в `views_channels.py` функцию, обрабатывающую POST-запрос приглашения (например, с маршрутом `POST /channels/<id>/invite`). В теле запроса ожидается идентификатор пользователя (например, поле формы `user_id`).

Логика:

1. Проверка входа и загрузка канала (если канала нет — 404).
2. Проверка прав: приглашать могут только владелец канала или админ; иначе 403.
3. Получить `user_id` из формы; проверить, что пользователь существует, активен (`archived_at IS NULL`) и ещё не в канале.
4. Определить роль: для канала с типом `read_only` — роль `read_only`, иначе — `member`.
5. Вставить запись в `channel_members`, сообщение в flash и редирект на страницу канала.

Пример реализации:

```python
def channel_invite_view(channel_id: int):
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

    # Приглашать могут только владелец или админ
    if ch["owner_id"] != user["id"] and not is_admin():
        conn.close()
        abort(403)

    invited_id = request.form.get("user_id")
    if not invited_id:
        conn.close()
        flash("Выберите пользователя.")
        return redirect(url_for("channel_view", channel_id=channel_id))

    try:
        invited_id = int(invited_id)
    except (TypeError, ValueError):
        conn.close()
        flash("Некорректный пользователь.")
        return redirect(url_for("channel_view", channel_id=channel_id))

    # Пользователь должен существовать, быть активным и не состоять в канале
    invited = conn.execute(
        "SELECT id FROM users WHERE id = ? AND archived_at IS NULL",
        (invited_id,),
    ).fetchone()
    if not invited:
        conn.close()
        flash("Пользователь не найден или архивирован.")
        return redirect(url_for("channel_view", channel_id=channel_id))

    existing = conn.execute(
        "SELECT 1 FROM channel_members WHERE channel_id = ? AND user_id = ?",
        (channel_id, invited_id),
    ).fetchone()
    if existing:
        conn.close()
        flash("Пользователь уже в канале.")
        return redirect(url_for("channel_view", channel_id=channel_id))

    role = "read_only" if ch["type"] == "read_only" else "member"
    conn.execute(
        "INSERT INTO channel_members (channel_id, user_id, role) VALUES (?, ?, ?)",
        (channel_id, invited_id, role),
    )
    conn.commit()
    conn.close()

    flash("Пользователь приглашён в канал.")
    return redirect(url_for("channel_view", channel_id=channel_id))
```

### Шаг H2.3. Маршрут и форма приглашения в шаблоне

В `app.py` добавьте импорт `channel_invite_view` и маршрут:

```python
from views_channels import (
    dashboard_view,
    channel_new_view,
    channel_create_view,
    channel_view_view,
    channel_join_view,
    channel_leave_view,
    channel_invite_view,
    channel_remove_member_view,
)
```

Маршрут для приглашения (можно разместить рядом с остальными маршрутами каналов):

```python
@app.post("/channels/<int:channel_id>/invite")
def channel_invite(channel_id):
    return channel_invite_view(channel_id)
```

В `templates/channel_view.html` внутри блока `{% if can_manage_members %}` добавьте форму приглашения **перед** таблицей участников (или после заголовка «Участники»):

```html
    {% if can_manage_members %}
      <h2>Участники</h2>
      <form method="post" action="{{ url_for('channel_invite', channel_id=channel.id) }}" class="form" style="margin-bottom: 16px;">
        <label for="user_id">Пригласить пользователя</label>
        <select id="user_id" name="user_id" required>
          <option value="">— выберите —</option>
          {% for u in invite_candidates %}
            <option value="{{ u.id }}">{{ u.username }}</option>
          {% endfor %}
        </select>
        <button type="submit" class="btn btn-primary">Пригласить</button>
      </form>
      {% if not invite_candidates %}
        <p class="muted">Нет пользователей для приглашения (все активные уже в канале).</p>
      {% endif %}
      <table>
        ...
      </table>
    {% endif %}
```

Если `invite_candidates` пуст, выпадающий список будет с одним пунктом «— выберите —»; можно не показывать кнопку «Пригласить» или выводить подсказку, как в примере выше.

---

## H3. Удаление участника из канала

Владелец канала или админ может удалить любого участника (в том числе другого владельца, если передали канал). Удаление — это удаление записи из `channel_members`. Если удалённый пользователь был **владельцем** канала, нужно применить ту же логику, что и при выходе владельца в Части G: передать владение другому участнику (например, с ролью `member` в приоритете) или обнулить `channels.owner_id`, если в канале никого не осталось.

### Шаг H3.1. Функция удаления участника

Добавьте в `views_channels.py` функцию, обрабатывающую POST-запрос вида `POST /channels/<id>/members/<user_id>/remove`:

1. Проверка входа, загрузка канала (404 при отсутствии).
2. Проверка прав: удалять могут только владелец канала или админ (403 иначе).
3. Проверить, что пользователь `user_id` действительно состоит в канале; иначе сообщение и редирект.
4. Удалить запись из `channel_members`.
5. Если удалённый пользователь был владельцем (`ch.owner_id == user_id`), выбрать нового владельца среди оставшихся участников (например, предпочитая роль `member`) и обновить `channels.owner_id` и роль в `channel_members`; если участников не осталось — установить `channels.owner_id = NULL`.
6. Flash-сообщение и редирект на страницу канала.

Пример:

```python
def channel_remove_member_view(channel_id: int, user_id: int):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    current = current_user()
    conn = get_conn()

    ch = conn.execute(
        "SELECT id, name, owner_id FROM channels WHERE id = ?",
        (channel_id,),
    ).fetchone()

    if ch is None:
        conn.close()
        abort(404)

    # Удалять участников могут только владелец или админ
    if ch["owner_id"] != current["id"] and not is_admin():
        conn.close()
        abort(403)

    member = conn.execute(
        "SELECT role FROM channel_members WHERE channel_id = ? AND user_id = ?",
        (channel_id, user_id),
    ).fetchone()

    if member is None:
        conn.close()
        flash("Пользователь не состоит в этом канале.")
        return redirect(url_for("channel_view", channel_id=channel_id))

    conn.execute(
        "DELETE FROM channel_members WHERE channel_id = ? AND user_id = ?",
        (channel_id, user_id),
    )

    # Если удалённый был владельцем — передаём владение или обнуляем owner_id
    if ch["owner_id"] == user_id:
        new_owner = conn.execute("""
            SELECT user_id FROM channel_members
            WHERE channel_id = ?
            ORDER BY role = 'member' DESC, user_id ASC
            LIMIT 1
        """, (channel_id,)).fetchone()

        if new_owner:
            conn.execute(
                "UPDATE channels SET owner_id = ? WHERE id = ?",
                (new_owner["user_id"], channel_id),
            )
            conn.execute(
                "UPDATE channel_members SET role = 'owner' WHERE channel_id = ? AND user_id = ?",
                (channel_id, new_owner["user_id"]),
            )
        else:
            conn.execute(
                "UPDATE channels SET owner_id = NULL WHERE id = ?",
                (channel_id,),
            )

    conn.commit()
    conn.close()

    flash("Участник удалён из канала.")
    return redirect(url_for("channel_view", channel_id=channel_id))
```

### Шаг H3.2. Маршрут для удаления

В `app.py` добавьте маршрут (импорт `channel_remove_member_view` уже указан в шаге H2.3):

```python
@app.post("/channels/<int:channel_id>/members/<int:user_id>/remove")
def channel_remove_member(channel_id, user_id):
    return channel_remove_member_view(channel_id, user_id)
```

В шаблоне кнопка «Удалить» в таблице участников уже ведёт на `url_for('channel_remove_member', channel_id=channel.id, user_id=m.user_id)` (см. шаг H1.2). Убедитесь, что форма отправляет POST на этот маршрут.

---

## H4. Итоговая структура блока «Участники» в шаблоне

Для единообразия ниже приведён полный фрагмент блока «Участники» в `channel_view.html`: заголовок, форма приглашения, подсказка при пустом списке кандидатов, таблица участников с кнопкой «Удалить».

```html
    {% if can_manage_members %}
      <h2>Участники</h2>
      <form method="post" action="{{ url_for('channel_invite', channel_id=channel.id) }}" class="form" style="margin-bottom: 16px;">
        <label for="user_id">Пригласить пользователя</label>
        <select id="user_id" name="user_id" required>
          <option value="">— выберите —</option>
          {% for u in invite_candidates %}
            <option value="{{ u.id }}">{{ u.username }}</option>
          {% endfor %}
        </select>
        <button type="submit" class="btn btn-primary">Пригласить</button>
      </form>
      {% if not invite_candidates %}
        <p class="muted">Нет пользователей для приглашения (все активные уже в канале).</p>
      {% endif %}
      <table>
        <thead>
          <tr>
            <th>Пользователь</th>
            <th>Роль</th>
            <th></th>
          </tr>
        </thead>
        <tbody>
          {% for m in members %}
            <tr>
              <td>{{ m.username }}</td>
              <td>{{ m.role }}</td>
              <td>
                <form method="post" action="{{ url_for('channel_remove_member', channel_id=channel.id, user_id=m.user_id) }}" style="display: inline;">
                  <button type="submit" class="btn btn-small">Удалить</button>
                </form>
              </td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    {% endif %}
```

Разместите этот блок после блока с «Ваша роль» / «Вступить» / «Покинуть канал» и перед заголовком «Сообщения».

---

## Итог Части H (Slack)

После выполнения этой части в проекте есть:

- **`views_channels.py`**: в `channel_view_view()` — выборка списка участников канала (`members`) и флаг `can_manage_members` (владелец или админ); при `can_manage_members` — выборка списка кандидатов для приглашения (`invite_candidates`: активные пользователи, не состоящие в канале). Функции `channel_invite_view()` (приглашение по `user_id` из формы, роль по типу канала) и `channel_remove_member_view()` (удаление из `channel_members`, при удалении владельца — передача владения или обнуление `owner_id`).
- **`app.py`**: маршруты `POST /channels/<id>/invite` и `POST /channels/<id>/members/<user_id>/remove` и соответствующие импорты.
- **`templates/channel_view.html`**: блок «Участники», видимый только при `can_manage_members`: форма приглашения (выпадающий список по `invite_candidates`), таблица участников с ролями и кнопкой «Удалить» для каждого.

Владелец канала и админ видят список участников, могут приглашать активных пользователей и удалять участников; при удалении владельца применяется та же логика, что и при его выходе в Части G. В следующей части можно переходить к сообщениям и тредам.
{% endraw %}
