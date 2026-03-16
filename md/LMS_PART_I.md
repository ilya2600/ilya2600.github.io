{% raw %}
# Часть I: Домашние задания — создание, просмотр, сдача и оценка

В Части H вы добавили занятия и посещаемость. На странице занятия есть блок «Домашние задания» и ссылка «Добавить домашнее задание» (маршрут был заглушкой). В этой части вы реализуете домашние задания: таблицы в БД, создание задания по занятию, страницу задания, сдачу работы учеником (один раз до дедлайна) и выставление оценки с комментарием учителем или менеджером.

В этой части вы добавляете:

- таблицы `homework` и `submissions` в базе;
- модуль `views_homework.py` с созданием задания, просмотром задания, сдачей и сохранением оценок;
- обновление `views_lessons.py`: на странице занятия загружать список домашних заданий из БД;
- маршруты для формы создания, просмотра задания, сдачи и сохранения оценок;
- шаблоны формы создания задания, страницы задания (для ученика — форма сдачи или «Ваша сдача»; для учителя/менеджера — список сдач с полями оценки и комментария);
- замену заглушки «Добавить домашнее задание» на реальные маршруты и обновление шаблона занятия при необходимости.

Прогресс ученика и общая статистика остаются на Часть J.

---

## Видимо / проверяемо

После выполнения Части I:

- На странице занятия отображается список домашних заданий; у того, кто может управлять занятием, есть ссылка «Добавить домашнее задание». После создания задание появляется в списке и открывается по ссылке.
- Страница задания показывает название, описание, ссылку (url), срок сдачи и ссылку на занятие.
- Ученик (из класса этого занятия) видит форму сдачи (текст и необязательная ссылка) или блок «Ваша сдача» с датой; после дедлайна сдача недоступна. Сдать можно только один раз.
- Учитель или менеджер на странице задания видит список всех сдач; может указать оценку (0–100) и комментарий по каждой сдаче и сохранить. Ученик после этого видит свою оценку и комментарий на странице задания.

---

## I1. Таблицы `homework` и `submissions` в `db.py`

### Шаг I1.1. Таблица `homework`

Откройте `db.py`, найдите функцию `init_db()`. Сразу **после** блока `CREATE TABLE IF NOT EXISTS attendance (...)` добавьте создание таблицы `homework`:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS homework (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            lesson_id INTEGER NOT NULL REFERENCES lessons(id) ON DELETE CASCADE,
            title TEXT NOT NULL,
            description TEXT,
            url TEXT,
            due_date TEXT NOT NULL,
            created_by INTEGER REFERENCES users(id) ON DELETE SET NULL,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        )
    """)
```

Поля: занятие, название, описание и ссылка (по желанию), срок сдачи (текст в формате даты/времени, например ISO), кто создал, дата создания.

### Шаг I1.2. Таблица `submissions`

Сразу после создания таблицы `homework` добавьте таблицу сдач:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS submissions (
            student_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
            homework_id INTEGER NOT NULL REFERENCES homework(id) ON DELETE CASCADE,
            text TEXT,
            url TEXT,
            grade INTEGER,
            feedback TEXT,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now')),
            PRIMARY KEY (student_id, homework_id)
        )
    """)
```

Одна запись на пару (ученик, задание): текст сдачи, необязательная ссылка, оценка (0–100, выставляет учитель/менеджер), комментарий.

---

## I2. Модуль `views_homework.py`

Создайте файл `views_homework.py` в той же папке, где лежат `app.py`, `views_lessons.py`.

### Шаг I2.1. Импорты и хелпер доступа к заданию

Видеть страницу задания и сдавать работу могут: ученики, входящие в класс этого занятия; пользователи, которые могут управлять занятием (учитель занятия, менеджер, мастер). Создавать задание могут только те, кто может управлять занятием.

В начало файла вставьте:

```python
from flask import render_template, redirect, url_for, request, abort
from db import get_conn
from auth_utils import is_logged_in, current_user, is_manager, is_master
from views_lessons import can_manage_lesson
```

Циклический импорт: `views_lessons` импортирует представления занятий и `can_manage_lesson`. Если импорт `views_lessons` из `views_homework` вызовет цикл (например, если `views_lessons` импортирует что‑то из `views_homework`), перенесите `can_manage_lesson` в общий модуль (например, `auth_utils` или отдельный `helpers.py`) и импортируйте оттуда. В текущем проекте `views_lessons` не импортирует `views_homework`, поэтому импорт `can_manage_lesson` из `views_lessons` допустим.

Функция проверки «может ли текущий пользователь видеть это задание» (ученик в классе занятия или может управлять занятием):

```python
def can_view_homework(lesson_row):
    """Может ли текущий пользователь видеть задание (страница и сдача)."""
    if not is_logged_in():
        return False
    u = current_user()
    if u is None:
        return False
    if can_manage_lesson(lesson_row):
        return True
    if u["role"] == "student":
        conn = get_conn()
        in_class = conn.execute(
            "SELECT 1 FROM class_students WHERE class_id = ? AND student_id = ?",
            (lesson_row["class_id"], u["id"]),
        ).fetchone()
        conn.close()
        return in_class is not None
    return False
```

### Шаг I2.2. Создание задания `homework_new_view(lesson_id)` и `homework_create_view(lesson_id)`

Создавать задание может только тот, кто может управлять занятием. Форма: название, описание, url, срок сдачи.

```python
def homework_new_view(lesson_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    conn = get_conn()
    lesson = conn.execute(
        "SELECT id, class_id, teacher_id, title FROM lessons WHERE id = ?",
        (lesson_id,),
    ).fetchone()
    if lesson is None:
        conn.close()
        abort(404)
    if not can_manage_lesson(lesson):
        conn.close()
        abort(403)
    conn.close()
    return render_template("homework_new.html", lesson_id=lesson_id, lesson=lesson, error=None)


def homework_create_view(lesson_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    conn = get_conn()
    lesson = conn.execute(
        "SELECT id, class_id, teacher_id, title FROM lessons WHERE id = ?",
        (lesson_id,),
    ).fetchone()
    if lesson is None:
        conn.close()
        abort(404)
    if not can_manage_lesson(lesson):
        conn.close()
        abort(403)

    title = (request.form.get("title") or "").strip()
    due_date = (request.form.get("due_date") or "").strip()
    description = (request.form.get("description") or "").strip()
    url = (request.form.get("url") or "").strip() or None

    if not title or not due_date:
        conn.close()
        return render_template(
            "homework_new.html",
            lesson_id=lesson_id,
            lesson=lesson,
            error="Укажите название и срок сдачи.",
        )

    u = current_user()
    conn.execute(
        """INSERT INTO homework (lesson_id, title, description, url, due_date, created_by)
           VALUES (?, ?, ?, ?, ?, ?)""",
        (lesson_id, title, description or None, url, due_date, u["id"]),
    )
    conn.commit()
    conn.close()
    return redirect(url_for("lesson_show", lesson_id=lesson_id))
```

В шаблон передаются `lesson_id`, `lesson` (с полями `id`, `class_id`, `teacher_id`, `title`) и при ошибке валидации — `error`.

### Шаг I2.3. Просмотр задания `homework_show_view(homework_id)`

Загрузить задание и занятие; проверить право через `can_view_homework(lesson)`; для ученика — проверить, есть ли уже сдача, и можно ли ещё сдавать (текущее время не позже срока сдачи); для учителя/менеджера — список всех сдач с полями для оценки и комментария.

```python
def homework_show_view(homework_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    conn = get_conn()
    homework = conn.execute(
        """SELECT h.id, h.lesson_id, h.title, h.description, h.url, h.due_date, h.created_by
           FROM homework h WHERE h.id = ?""",
        (homework_id,),
    ).fetchone()
    if homework is None:
        conn.close()
        abort(404)

    lesson = conn.execute(
        "SELECT id, class_id, teacher_id FROM lessons WHERE id = ?",
        (homework["lesson_id"],),
    ).fetchone()
    if lesson is None:
        conn.close()
        abort(404)

    if not can_view_homework(lesson):
        conn.close()
        abort(403)

    lesson_title_row = conn.execute(
        "SELECT title FROM lessons WHERE id = ?",
        (homework["lesson_id"],),
    ).fetchone()
    lesson_title = lesson_title_row["title"] if lesson_title_row else ""

    u = current_user()
    can_manage = can_manage_lesson(lesson)
    my_submission = None
    submissions_list = []
    can_still_submit = False

    if u["role"] == "student":
        my_submission = conn.execute(
            "SELECT text, url, grade, feedback, created_at FROM submissions WHERE homework_id = ? AND student_id = ?",
            (homework_id, u["id"]),
        ).fetchone()
        can_still_submit = my_submission is None
        if can_still_submit and homework["due_date"]:
            from datetime import datetime, timezone
            try:
                due_dt = datetime.fromisoformat(homework["due_date"].replace("Z", "+00:00"))
                if due_dt.tzinfo is None:
                    due_dt = due_dt.replace(tzinfo=timezone.utc)
                can_still_submit = datetime.now(timezone.utc) <= due_dt
            except Exception:
                pass
    elif can_manage:
        submissions_list = conn.execute(
            """SELECT s.student_id, s.text, s.url, s.grade, s.feedback, s.created_at, u.username
               FROM submissions s
               JOIN users u ON u.id = s.student_id
               WHERE s.homework_id = ?
               ORDER BY u.username""",
            (homework_id,),
        ).fetchall()

    conn.close()

    return render_template(
        "homework_show.html",
        homework=homework,
        lesson_title=lesson_title,
        can_manage=can_manage,
        my_submission=my_submission,
        can_still_submit=can_still_submit,
        submissions_list=submissions_list,
    )
```

Срок сдачи храните в `homework.due_date` как текст (например, ISO). Для проверки «ещё можно сдать» парсите его в `datetime` и сравнивайте с текущим временем (UTC). При ошибке парсинга можно считать, что сдавать ещё можно.

### Шаг I2.4. Сдача работы `homework_submit_view(homework_id)`

Принимает POST: текст и необязательная ссылка. Только ученик, один раз на задание, только до дедлайна.

```python
def homework_submit_view(homework_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    u = current_user()
    if u is None or u["role"] != "student":
        abort(403)

    conn = get_conn()
    homework = conn.execute("SELECT id, lesson_id, due_date FROM homework WHERE id = ?", (homework_id,)).fetchone()
    if homework is None:
        conn.close()
        abort(404)
    lesson = conn.execute("SELECT id, class_id, teacher_id FROM lessons WHERE id = ?", (homework["lesson_id"],)).fetchone()
    if lesson is None or not can_view_homework(lesson):
        conn.close()
        abort(403)
    existing = conn.execute(
        "SELECT 1 FROM submissions WHERE homework_id = ? AND student_id = ?",
        (homework_id, u["id"]),
    ).fetchone()
    if existing:
        conn.close()
        return redirect(url_for("homework_show", homework_id=homework_id))
    from datetime import datetime, timezone
    if homework["due_date"]:
        try:
            due_dt = datetime.fromisoformat(homework["due_date"].replace("Z", "+00:00"))
            if due_dt.tzinfo is None:
                due_dt = due_dt.replace(tzinfo=timezone.utc)
            if datetime.now(timezone.utc) > due_dt:
                conn.close()
                return redirect(url_for("homework_show", homework_id=homework_id))
        except Exception:
            pass
    text = (request.form.get("text") or "").strip()
    url = (request.form.get("url") or "").strip() or None
    conn.execute(
        "INSERT INTO submissions (student_id, homework_id, text, url) VALUES (?, ?, ?, ?)",
        (u["id"], homework_id, text or None, url),
    )
    conn.commit()
    conn.close()
    return redirect(url_for("homework_show", homework_id=homework_id))
```

### Шаг I2.5. Сохранение оценок `homework_grades_save_view(homework_id)`

Доступно тому, кто может управлять занятием. В форме для каждой сдачи передаются поля вида `grade_<student_id>` и `feedback_<student_id>`. Обновить записи в `submissions`.

```python
def homework_grades_save_view(homework_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    conn = get_conn()
    homework = conn.execute("SELECT id, lesson_id FROM homework WHERE id = ?", (homework_id,)).fetchone()
    if homework is None:
        conn.close()
        abort(404)
    lesson = conn.execute("SELECT id, class_id, teacher_id FROM lessons WHERE id = ?", (homework["lesson_id"],)).fetchone()
    if lesson is None or not can_manage_lesson(lesson):
        conn.close()
        abort(403)
    rows = conn.execute(
        "SELECT student_id FROM submissions WHERE homework_id = ?",
        (homework_id,),
    ).fetchall()
    for row in rows:
        sid = row["student_id"]
        grade_val = request.form.get(f"grade_{sid}", type=int)
        feedback_val = (request.form.get(f"feedback_{sid}") or "").strip() or None
        if grade_val is not None and 0 <= grade_val <= 100:
            conn.execute(
                "UPDATE submissions SET grade = ?, feedback = ? WHERE homework_id = ? AND student_id = ?",
                (grade_val, feedback_val, homework_id, sid),
            )
    conn.commit()
    conn.close()
    return redirect(url_for("homework_show", homework_id=homework_id))
```

---

## I3. Обновление `views_lessons.py`: загрузка списка домашних заданий

В функции `lesson_show_view()` замените присваивание `homework_list = []` на выборку из БД: все задания по этому занятию (например, по `lesson_id`), чтобы передать в шаблон список с полями `id`, `title` (и при необходимости `due_date`). После загрузки `teacher_name` и перед формированием `can_manage` выполните, например:

```python
    homework_list = conn.execute(
        "SELECT id, title, due_date FROM homework WHERE lesson_id = ? ORDER BY created_at",
        (lesson_id,),
    ).fetchall()
```

Переменную `homework_list` по‑прежнему передавайте в `render_template("lesson_show.html", ...)`.

---

## I4. Маршруты в `app.py`

### Шаг I4.1. Импорт

Добавьте импорт из `views_homework`:

```python
from views_homework import (
    homework_new_view,
    homework_create_view,
    homework_show_view,
    homework_submit_view,
    homework_grades_save_view,
)
```

### Шаг I4.2. Замена заглушки и добавление маршрутов

Удалите заглушку для GET `/lessons/<int:lesson_id>/homework/new` (вызов `abort(404)`). Добавьте маршруты:

- GET `/lessons/<int:lesson_id>/homework/new` — форма создания задания; обработчик вызывает `homework_new_view(lesson_id)`.
- POST `/lessons/<int:lesson_id>/homework/new` — создание задания; обработчик вызывает `homework_create_view(lesson_id)`.
- GET `/homework/<int:homework_id>` — страница задания; обработчик вызывает `homework_show_view(homework_id)`.
- POST `/homework/<int:homework_id>/submit` — сдача работы учеником; обработчик вызывает `homework_submit_view(homework_id)`.
- POST `/homework/<int:homework_id>/grades` — сохранение оценок и комментариев; обработчик вызывает `homework_grades_save_view(homework_id)`.

Имена маршрутов (для `url_for`): `homework_new`, `homework_create`, `homework_show`, `homework_submit`, `homework_grades_save`. Убедитесь, что в шаблонах используются именно они (например, форма сдачи отправляется на `url_for('homework_submit', homework_id=homework.id)`).

---

## I5. Шаблоны

### Шаг I5.1. Форма создания задания `templates/homework_new.html`

Создайте шаблон с формой: название, описание (textarea), url, срок сдачи (например, `input type="datetime-local"` или `type="date"` и при необходимости время отдельно), кнопка «Создать». Метод POST, действие — `url_for('homework_create', lesson_id=lesson_id)`. Вывод ошибки из переменной `error`. Ссылка «К занятию» на `url_for('lesson_show', lesson_id=lesson_id)`.

### Шаг I5.2. Страница задания `templates/homework_show.html`

Создайте шаблон страницы задания.

- Общие блоки: название задания, описание, ссылка (если есть), срок сдачи, ссылка на занятие (`url_for('lesson_show', lesson_id=homework.lesson_id)` с текстом, например, `lesson_title`).
- Для ученика (`my_submission` и `can_still_submit`):
  - если `can_still_submit`: форма с полями «Текст» и «Ссылка» (url), метод POST, действие `url_for('homework_submit', homework_id=homework.id)`;
  - иначе: блок «Ваша сдача» — текст, ссылка (если есть), дата сдачи; если выставлены оценка и комментарий — вывести их.
- Для учителя/менеджера (`can_manage` и `submissions_list`): таблица (или список) сдач: ученик, текст, ссылка, дата, текущие оценка и комментарий, поля ввода «Оценка (0–100)» и «Комментарий» для каждой строки; одна форма на все строки, метод POST, действие `url_for('homework_grades_save', homework_id=homework.id)`, имена полей `grade_<student_id>`, `feedback_<student_id>`.

Пример структуры (без полного HTML):

```html
{% extends "base.html" %}
{% block title %}{{ homework.title }} · Мини‑ЛМС{% endblock %}
{% block content %}
  <section class="card">
    <h1>{{ homework.title }}</h1>
    <p><a href="{{ url_for('lesson_show', lesson_id=homework.lesson_id) }}">← К занятию: {{ lesson_title }}</a></p>
    {% if homework.description %}<p>{{ homework.description }}</p>{% endif %}
    {% if homework.url %}<p>Ссылка: <a href="{{ homework.url }}">{{ homework.url }}</a></p>{% endif %}
    <p><strong>Срок сдачи:</strong> {{ homework.due_date[:19] if homework.due_date else '' }}</p>

    {% if can_still_submit %}
      <h2>Сдать работу</h2>
      <form method="post" action="{{ url_for('homework_submit', homework_id=homework.id) }}">
        <div class="form-group">
          <label for="text">Текст</label>
          <textarea id="text" name="text" rows="4"></textarea>
        </div>
        <div class="form-group">
          <label for="url">Ссылка (по желанию)</label>
          <input type="url" id="url" name="url" />
        </div>
        <button type="submit" class="btn btn-primary">Отправить</button>
      </form>
    {% elif my_submission %}
      <h2>Ваша сдача</h2>
      <p>{{ my_submission.text or '—' }}</p>
      {% if my_submission.url %}<p>Ссылка: <a href="{{ my_submission.url }}">{{ my_submission.url }}</a></p>{% endif %}
      <p>Дата: {{ my_submission.created_at[:19] if my_submission.created_at else '' }}</p>
      {% if my_submission.grade is not none %}
        <p><strong>Оценка:</strong> {{ my_submission.grade }}</p>
        {% if my_submission.feedback %}<p><strong>Комментарий:</strong> {{ my_submission.feedback }}</p>{% endif %}
      {% endif %}
    {% endif %}

    {% if can_manage and submissions_list %}
      <h2>Сдачи и оценки</h2>
      <form method="post" action="{{ url_for('homework_grades_save', homework_id=homework.id) }}">
        <table class="table">
          <thead>
            <tr>
              <th>Ученик</th>
              <th>Текст / ссылка</th>
              <th>Дата</th>
              <th>Оценка (0–100)</th>
              <th>Комментарий</th>
            </tr>
          </thead>
          <tbody>
            {% for s in submissions_list %}
              <tr>
                <td>{{ s.username }}</td>
                <td>{{ s.text or '—' }}{% if s.url %} <a href="{{ s.url }}">ссылка</a>{% endif %}</td>
                <td>{{ s.created_at[:19] if s.created_at else '' }}</td>
                <td><input type="number" name="grade_{{ s.student_id }}" value="{{ s.grade if s.grade is not none else '' }}" min="0" max="100" /></td>
                <td><input type="text" name="feedback_{{ s.student_id }}" value="{{ s.feedback or '' }}" /></td>
              </tr>
            {% endfor %}
          </tbody>
        </table>
        <button type="submit" class="btn btn-primary">Сохранить оценки</button>
      </form>
    {% endif %}
  </section>
{% endblock %}
```

В шаблоне занятия (`lesson_show.html`) ссылка «Добавить домашнее задание» уже ведёт на `url_for('homework_new', lesson_id=lesson.id)`. После реализации маршрутов заглушку убирают; текст «(маршрут будет в Части I)» из ссылки можно удалить.

---

## Итог Части I

После выполнения части в проекте есть:

- **`db.py`**: таблицы `homework` и `submissions` в `init_db()`.
- **`views_homework.py`**: функции `can_view_homework(lesson_row)`, `homework_new_view`, `homework_create_view`, `homework_show_view`, `homework_submit_view`, `homework_grades_save_view`; проверка срока сдачи и единственной сдачи на ученика.
- **`views_lessons.py`**: в `lesson_show_view()` список домашних заданий загружается из БД и передаётся в шаблон.
- **`app.py`**: маршруты GET/POST для создания задания по занятию, GET страницы задания, POST сдачи и POST сохранения оценок; заглушка для «Новое домашнее задание» заменена на реальные обработчики.
- **Шаблоны**: `homework_new.html`, `homework_show.html`; при необходимости обновлён `lesson_show.html` (убрана пометка про Часть I).

Ученик может один раз сдать работу до дедлайна и затем видеть оценку и комментарий; учитель и менеджер выставляют оценки и комментарии на странице задания. Данные готовы для Части J (прогресс и статистика).
{% endraw %}
