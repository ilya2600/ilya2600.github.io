{% raw %}
# Часть J: История заявок и архивирование проектов

В Частях G–I вы уже сделали:

- **G** — проекты и привязка заявок к проектам:
  - таблица `projects` с полями `owner_id`, `title`, `description`, `is_archived`, `created_at`;
  - таблица `tickets` с полем `project_id` и маршруты `/projects/<project_id>/tickets/...`;
  - страницы списка проектов, создания и просмотра проекта;
  - раздел «Заявки этого проекта» на странице проекта.

- **H** — участники проектов и роли:
  - таблица `project_members` (`owner`, `maintainer`, `reporter`);
  - модуль `project_members.py` с хелперами:
    - `get_project_role(project_id)`,
    - `can_view_project(project_id)`,
    - `can_manage_members(project_id)`,
    - `can_create_ticket(project_id)`,
    - `can_edit_ticket_in_project(ticket_row)`;
  - раздел «Участники проекта» и форма добавления участника (выпадающий список по username);
  - разделение `/projects` на «Мои проекты» и «Проекты, где я участвую».

- **I** — комментарии к заявкам:
  - таблица `comments` (`ticket_id`, `user_id`, `body`, `created_at`);
  - вывод комментариев в `ticket_detail_view` и `ticket_detail.html`;
  - форма добавления комментария с POST на `/projects/<project_id>/tickets/<ticket_id>/comments`;
  - базовая проверка `can_comment`:
    - пользователь залогинен,
    - пользователь участник проекта (`can_view_project`).

В этой финальной части J мы:

- добавим **историю изменений заявок** (таблица `issue_history`);
- начнём писать туда события:
  - создание заявки,
  - изменение основных полей (статус, приоритет, исполнитель и т.п., если вы уже добавили такую логику),
  - добавление комментария;
- реализуем **архивирование проектов**:
  - archive / restore / delete для проекта,
  - режим read‑only для архивированных проектов (заявки и комментарии).

> Важно: в вашем текущем коде заявки пока, скорее всего, ещё не поддерживают изменения
> статуса / исполнителя / редактирование описания через отдельные формы (это может быть
> частью дополнительных упражнений). В Части J мы покажем, **куда** подключать историю,
> а минимально необходимый лог — это запись о создании заявки и о добавлении комментариев.

---

## J0. Актуальное состояние перед началом

Перед выполнением этой части у вас уже должны быть:

- таблицы `projects`, `tickets`, `project_members`, `comments` в `db.py`;
- поле `is_archived` в таблице `projects` (Часть G);
- модуль `project_members.py` и проектные роли;
- `views_projects.py` и `views_tickets.py`, реализующие:
  - просмотр проектов и заявок только для участников (`can_view_project` / `can_view_ticket`);
  - добавление комментариев к заявкам;
  - пока **нет** явной логики архивирования / восстановления / удаления проектов.

В Части J мы:

- **добавим** таблицу `issue_history`;
- **обновим** часть логики заявок и комментариев, чтобы писать события в историю;
- **реализуем** архивирование / восстановление / удаление проекта;
- **запретим изменения** заявок и комментариев в архивных проектах (read‑only).

> Рекомендация: как и в Частях G–I, удобнее всего удалить `database.db` и пересоздать
> базу после того, как вы добавите таблицу `issue_history` и обновите логику архивирования,
> чтобы не получать ошибки из‑за старой схемы.

---

## J1. Таблица `issue_history` в `db.py`

Первый шаг — добавить таблицу для истории заявок.

### J1.1. Добавляем `issue_history`

Откройте `db.py` и найдите функцию:

```python
def init_db():
    ...
```

Сразу **после** создания таблицы `comments` (или рядом с ней) добавьте:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS issue_history (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            ticket_id INTEGER NOT NULL,
            user_id INTEGER,
            action_type TEXT NOT NULL,
            data TEXT,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            FOREIGN KEY (ticket_id) REFERENCES tickets(id),
            FOREIGN KEY (user_id) REFERENCES users(id)
        )
    """)
```

Поля:

- `ticket_id` — к какой заявке относится событие;
- `user_id` — кто совершил действие (может быть `NULL` для системных событий);
- `action_type` — тип события (`'created'`, `'comment_added'`, `'status_changed'` и т.п.);
- `data` — произвольная строка с данными (например, JSON или краткое описание);
- `created_at` — время записи события.

---

## J2. Хелпер для записи истории

Чтобы не дублировать код вставки в `issue_history`, вынесем его в отдельную функцию.

### J2.1. Новый модуль `history_utils.py`

В корне проекта (рядом с `app.py`, `views_tickets.py`, `views_projects.py`) создайте файл:

- `history_utils.py`

Откройте его и вставьте:

```python
import json

from db import get_conn
from auth_utils import current_user


def log_issue_event(ticket_id, action_type, data_dict=None):
    """Записать событие в таблицу issue_history."""
    user = current_user()
    user_id = user["id"] if user is not None else None
    data_str = json.dumps(data_dict or {}, ensure_ascii=False)

    conn = get_conn()
    conn.execute(
        """
        INSERT INTO issue_history (ticket_id, user_id, action_type, data)
        VALUES (?, ?, ?, ?)
        """,
        (ticket_id, user_id, action_type, data_str),
    )
    conn.commit()
    conn.close()
```

Пояснения:

- функция сама получает текущего пользователя через `current_user()`;
- `data_dict` сериализуется в JSON и попадает в поле `data`;
- если данных нет, записываем `{}`.

> Если вы не хотите тянуть `json`, можно вместо этого хранить простой текст
> (`data_str = data_dict or ""`), но для примеров в Части J мы будем считать,
> что формат JSON допустим.

---

## J3. Пишем историю при создании заявки и добавлении комментария

Теперь подключим `log_issue_event` к существующим действиям.

### J3.1. История при создании заявки

Откройте `views_tickets.py` и в начало файла добавьте импорт:

```python
from history_utils import log_issue_event
```

Найдите функцию:

```python
def ticket_create_view(project_id):
    ...
```

Сейчас она:

- проверяет логин и `can_create_ticket`,
- валидирует `title`, `description`, `priority`,
- вставляет запись в таблицу `tickets`,
- редиректит на список заявок проекта.

Нам нужен `id` созданной заявки, чтобы записать событие.

Обновите вставку в `tickets` так, чтобы получить `lastrowid`:

```python
    conn = get_conn()
    cursor = conn.execute(
        "INSERT INTO tickets (title, description, reporter_id, project_id, status, priority) "
        "VALUES (?, ?, ?, ?, 'Open', ?)",
        (title, description, user["id"], project_id, priority),
    )
    ticket_id = cursor.lastrowid
    conn.commit()
    conn.close()

    log_issue_event(
        ticket_id,
        "created",
        {
            "project_id": project_id,
            "title": title,
            "priority": priority,
        },
    )

    return redirect(url_for("ticket_list", project_id=project_id))
```

Так в истории появится запись о создании заявки с основными полями.

### J3.2. История при добавлении комментария

В `views_tickets.py` найдите функцию:

```python
def comment_create_view(project_id: int, ticket_id: int):
    ...
```

После успешной вставки в таблицу `comments` добавьте вызов `log_issue_event`:

```python
    conn.execute(
        """
        INSERT INTO comments (ticket_id, user_id, body)
        VALUES (?, ?, ?)
        """,
        (ticket_id, user_id, body),
    )
    conn.commit()
    conn.close()

    log_issue_event(
        ticket_id,
        "comment_added",
        {
            "project_id": project_id,
            "snippet": body[:100],
        },
    )

    return redirect(url_for("ticket_detail", project_id=project_id, ticket_id=ticket_id))
```

Теперь каждый новый комментарий будет создавать событие в истории.

> При желании вы можете добавить аналогичные вызовы `log_issue_event` в местах,
> где будете реализовывать смену статуса, исполнителя или редактирование заявки.

---

## J4. Отображение истории в `ticket_detail_view` и `ticket_detail.html`

Следующий шаг — показывать историю на странице заявки.

### J4.1. Загружаем историю в `ticket_detail_view`

В `views_tickets.py` в функции `ticket_detail_view` после выборки `comments` добавьте:

```python
    history = conn.execute(
        """
        SELECT
            h.id,
            h.action_type,
            h.data,
            h.created_at,
            h.user_id,
            u.username AS author_username
        FROM issue_history h
        LEFT JOIN users u ON h.user_id = u.id
        WHERE h.ticket_id = ?
        ORDER BY h.created_at DESC
        """,
        (ticket_id,),
    ).fetchall()
```

и перед закрытием соединения не забудьте добавить `history` в контекст рендера:

```python
    conn.close()

    user = current_user()
    can_comment = (
        user is not None
        and can_view_project(project_id)
    )

    return render_template(
        "ticket_detail.html",
        ticket=t,
        can_edit=can_edit_ticket(ticket_id),
        project_id=project_id,
        comments=comments,
        can_comment=can_comment,
        history=history,
    )
```

### J4.2. Раздел «История» в `ticket_detail.html`

Откройте `templates/ticket_detail.html`.

Под секцией с комментариями (или рядом с ней) добавьте новый блок:

```html
  <section class="card" style="margin-top: 16px;">
    <h3>История</h3>

    {% if not history %}
      <p class="muted">Для этой заявки пока нет записей в истории.</p>
    {% else %}
      <ul class="history-list">
        {% for h in history %}
          <li class="history-item">
            <div class="history-meta">
              <span class="history-date">
                {{ h.created_at[:19] if h.created_at else '' }}
              </span>
              <span class="history-user">
                {{ h.author_username or 'System' }}
              </span>
            </div>
            <div class="history-description">
              {% if h.action_type == 'created' %}
                Заявка создана.
              {% elif h.action_type == 'comment_added' %}
                Добавлен комментарий.
              {% elif h.action_type == 'status_changed' %}
                Статус заявки изменён.
              {% else %}
                Действие: {{ h.action_type }}.
              {% endif %}
            </div>
          </li>
        {% endfor %}
      </ul>
    {% endif %}
  </section>
```

Здесь мы интерпретируем `action_type` в простые человеко‑читаемые фразы.
Поле `data` пока не используется в шаблоне (его можно задействовать позже для более
подробного описания изменений).

---

## J5. Архивирование, восстановление и удаление проектов

Теперь реализуем жизненный цикл проекта: архив, восстановление и удаление.

### J5.1. Маршруты в `views_projects.py`

Откройте `views_projects.py`.

В начало файла добавьте (если ещё нет) импорт `flash` и функции `is_logged_in` / `current_user` уже есть:

```python
from flask import render_template, redirect, url_for, request, abort, flash
```

Ниже существующих функций (`project_detail_view`, `project_member_add_view`, `project_member_remove_view`)
добавьте три новых обработчика:

```python
def project_archive_view(project_id):
    """Заархивировать проект (owner)."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    project = get_project(project_id)
    if project is None:
        abort(404)

    if not can_manage_members(project_id):
        abort(403)

    conn = get_conn()
    conn.execute(
        "UPDATE projects SET is_archived = 1 WHERE id = ?",
        (project_id,),
    )
    conn.commit()
    conn.close()

    flash("Проект отправлен в архив. Заявки и комментарии стали только для чтения.")
    return redirect(url_for("project_detail", project_id=project_id))


def project_restore_view(project_id):
    """Восстановить проект из архива (owner)."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    project = get_project(project_id)
    if project is None:
        abort(404)

    if not can_manage_members(project_id):
        abort(403)

    conn = get_conn()
    conn.execute(
        "UPDATE projects SET is_archived = 0 WHERE id = ?",
        (project_id,),
    )
    conn.commit()
    conn.close()

    flash("Проект восстановлен из архива.")
    return redirect(url_for("project_detail", project_id=project_id))


def project_delete_view(project_id):
    """Полностью удалить проект и связанные данные (owner)."""
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    project = get_project(project_id)
    if project is None:
        abort(404)

    if not can_manage_members(project_id):
        abort(403)

    conn = get_conn()

    # Находим все заявки этого проекта
    ticket_ids = [
        row["id"]
        for row in conn.execute(
            "SELECT id FROM tickets WHERE project_id = ?",
            (project_id,),
        ).fetchall()
    ]

    if ticket_ids:
        placeholders = ",".join("?" for _ in ticket_ids)

        # Удаляем комментарии и историю для этих заявок
        conn.execute(
            f"DELETE FROM comments WHERE ticket_id IN ({placeholders})",
            ticket_ids,
        )
        conn.execute(
            f"DELETE FROM issue_history WHERE ticket_id IN ({placeholders})",
            ticket_ids,
        )

        # Удаляем сами заявки
        conn.execute(
            f"DELETE FROM tickets WHERE id IN ({placeholders})",
            ticket_ids,
        )

    # Удаляем участников проекта и сам проект
    conn.execute(
        "DELETE FROM project_members WHERE project_id = ?",
        (project_id,),
    )
    conn.execute(
        "DELETE FROM projects WHERE id = ?",
        (project_id,),
    )

    conn.commit()
    conn.close()

    flash("Проект и все связанные данные удалены безвозвратно.")
    return redirect(url_for("projects_list"))
```

> Обратите внимание на использование `placeholders` и `ticket_ids` — мы безопасно
> подставляем список ID через плейсхолдеры, а не конкатенируем строки вручную.

### J5.2. Маршруты в `app.py`

Откройте `app.py` и в блок импортов `from views_projects import ...` добавьте новые функции:

```python
from views_projects import (
    projects_list_view,
    project_new_view,
    project_create_view,
    project_detail_view,
    project_member_add_view,
    project_member_remove_view,
    project_archive_view,
    project_restore_view,
    project_delete_view,
)
```

Ниже уже существующего маршрута `project_detail` добавьте:

```python
@app.post("/projects/<int:project_id>/archive")
def project_archive(project_id):
    return project_archive_view(project_id)


@app.post("/projects/<int:project_id>/restore")
def project_restore(project_id):
    return project_restore_view(project_id)


@app.post("/projects/<int:project_id>/delete")
def project_delete(project_id):
    return project_delete_view(project_id)
```

---

## J6. Read‑only для архивированных проектов

Теперь нужно, чтобы архивированные проекты были **только для чтения**:

- нельзя создавать / изменять заявки;
- нельзя создавать / удалять комментарии;
- но просмотр остаётся доступным участникам.

### J6.1. Проверяем `is_archived` в `project_detail_view`

В `views_projects.py` в `project_detail_view` после загрузки проекта (`project = get_project(project_id)`)
и проверки `can_view_project` уже выполняется остальная логика.

Вы можете:

- либо просто передать `project["is_archived"]` в шаблон и отрисовывать там бейдж,
- либо дополнительно использовать это поле в других местах.

В `render_template` уже передаётся объект `project`, поэтому в шаблоне `project_detail.html`
достаточно добавить, например:

```html
    <p>
      <strong>Создан:</strong> {{ project.created_at }}
      {% if project.is_archived %}
        <span class="badge badge-deleted">В архиве</span>
      {% endif %}
    </p>
```

### J6.2. Блокируем POST‑действия при `is_archived = 1`

#### Заявки (`views_tickets.py`)

В `ticket_new_view` и `ticket_create_view` перед проверками `can_create_ticket` можно добавить:

```python
from db import get_conn
from views_projects import get_project  # если нужно
```

и внутри функции:

```python
    project = get_project(project_id)
    if project is None or project["is_archived"]:
        abort(403)
```

Аналогично, если у вас есть обработчики изменения заявок (смена статуса, исполнителя),
нужно в них добавлять ту же проверку.

#### Комментарии (`comment_create_view`)

В `comment_create_view` после проверки `can_view_ticket` добавьте:

```python
    project = get_project(project_id)
    if project is None or project["is_archived"]:
        abort(403)
```

Это гарантирует, что в архивный проект нельзя добавить новый комментарий.

#### Управление участниками (`views_projects.py`)

В `project_member_add_view`, `project_member_remove_view` имеет смысл тоже
запретить изменения при `is_archived = 1` (по желанию), добавив аналогичную проверку.

---

## J7. Кнопки «Архивировать», «Восстановить», «Удалить» на странице проекта

Чтобы пользователь мог использовать новые маршруты, добавим кнопки в `project_detail.html`.

Откройте `templates/project_detail.html` и в верхней карточке с информацией о проекте
добавьте (для владельца / owner’а):

```html
    <p style="margin-top: 12px;">
      <a href="{{ url_for('projects_list') }}">К списку проектов</a>
    </p>

    {% if project_role == 'owner' %}
      <form
        method="post"
        action="{{ url_for('project_archive', project_id=project.id) }}"
        class="inline-form"
      >
        {% if not project.is_archived %}
          <button
            type="submit"
            class="btn btn-sm"
            onclick="return confirm('Отправить проект в архив? Заявки и комментарии станут только для чтения.')"
          >
            Архивировать проект
          </button>
        {% else %}
          <button type="submit" class="btn btn-sm">Восстановить проект</button>
        {% endif %}
      </form>

      {% if project.is_archived %}
        <form
          method="post"
          action="{{ url_for('project_delete', project_id=project.id) }}"
          class="inline-form"
          style="margin-left: 8px;"
        >
          <button
            type="submit"
            class="btn btn-sm btn-danger"
            onclick="return confirm('Удалить проект и все связанные данные безвозвратно?')"
          >
            Удалить проект
          </button>
        </form>
      {% endif %}
    {% endif %}
```

Пояснения:

- кнопка «Архивировать» доступна только если проект ещё не в архиве;
- кнопка «Восстановить» — если `project.is_archived == 1`;
- кнопку «Удалить проект» (hard delete) показываем только для архивированных проектов,
  чтобы случайно не удалить активный проект.

---

## J8. Видимая / проверяемая функциональность

После реализации Части J проверьте:

1. **Создание заявки и комментариев — история.**
   - Создайте новый проект и заявку.
   - Откройте заявку — в разделе «История» должна быть запись «Заявка создана».
   - Добавьте комментарий — в истории должна появиться новая запись «Добавлен комментарий».

2. **Архивирование проекта.**
   - На странице проекта как `owner` нажмите «Архивировать проект».
   - В шапке проекта должен появиться бейдж «В архиве».
   - Попробуйте:
     - создать новую заявку в этом проекте,
     - добавить новый комментарий,
     - (если есть) изменить статус заявки.  
     Все такие действия должны давать 403 или быть заблокированы (нет кнопок).

3. **Восстановление проекта.**
   - Нажмите «Восстановить проект».
   - Убедитесь, что:
     - бейдж «В архиве» пропал,
     - создания заявок и комментариев снова работают.

4. **Удаление проекта.**
   - Заархивируйте проект.
   - Нажмите «Удалить проект» и подтвердите.
   - Проект должен пропасть из списков:
     - «Мои проекты» и «Проекты, где я участвую»,
     - попытка открыть `/projects/<id>` или связанные URL заявок должна приводить к 404 / редиректу.

5. **История после удаления.**
   - Для удалённых проектов:
     - заявки, комментарии и история по ним больше не должны быть доступны
       (мы удалили связанные записи в J5.1).

---

## J9. Замечания и возможные конфликты

- **Совместимость с текущей схемой.**  
  Добавление `issue_history` и проверок `is_archived` требует, чтобы таблица `projects`
  уже содержала поле `is_archived` (часть G) и чтобы все маршруты заявок были
  «проектными» (`/projects/<project_id>/tickets/...`). В вашем коде это уже так,
  поэтому инструкции Части J остаются последовательными.

- **Отсутствие сложного UI редактирования заявок.**  
  План `issue_parts_plan_v2.md` предполагает более богатый жизненный цикл заявок
  (смена статуса, приоритета, исполнителя и т.п.). В текущем учебном проекте
  некоторые из этих действий могут быть ещё не реализованы.  
  Часть J закладывает основу истории (`issue_history`) и показывает,
  куда дописывать логику истории по мере появления новых действий.

- **Удаление проекта и связанных данных.**  
  В `project_delete_view` мы сознательно:
  - удаляем все комментарии и историю, привязанные к заявкам проекта;
  - удаляем сами заявки;
  - удаляем участников и сам проект.  
  Это «сильное» поведение (никакого soft delete), но для учебного приложения упрощает
  понимание и не оставляет «висячих» записей в БД.

- **Блокировка действий в архивных проектах.**  
  Важно не ограничиваться скрытием кнопок в шаблонах. В Части J мы также добавляем
  проверки `project.is_archived` в обработчики POST (создание заявок / комментариев).
  Если вы забудете эту часть, пользователь сможет создать заявку / комментарий через
  прямой POST‑запрос, даже если в UI кнопки отсутствуют.

После частей G–J ваше учебное приложение реализует базовую модель трекера задач:

- проекты с участниками и ролями,
- заявки и комментарии в контексте проектов,
- простую историю событий,
- архивирование и удаление проектов.

Дальше вы можете усложнять модель:

- добавлять редактирование заявок,
- хранить в `data` подробные дельты изменений,
- улучшать UI истории и статуса проектов.

{% endraw %}

