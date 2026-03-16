{% raw %}
# Часть H: Занятия — список, создание, просмотр и посещаемость

В Части G вы добавили просмотр класса и привязку учеников к классам. Теперь нужны занятия (уроки): таблица занятий, список по ролям, создание занятия менеджером или мастером, страница занятия с данными и посещаемостью.

В этой части вы добавляете:

- таблицы `lessons` и `attendance` в базе;
- хелперы `is_teacher()` и `can_manage_lesson()` для проверки прав;
- модуль `views_lessons.py` со списком занятий (с фильтром по роли), созданием и просмотром занятия, сохранением посещаемости;
- маршруты для занятий и посещаемости;
- шаблоны списка, формы создания и страницы занятия; блок посещаемости на странице занятия для учителя/менеджера/мастера;
- пункты навигации «Занятия» и «Новое занятие».

На странице занятия выводится блок «Домашние задания» (пока пустой) и ссылка «Добавить домашнее задание»; маршрут для этой ссылки будет добавлен в Части I.

---

## Видимо / проверяемо

После выполнения Части H:

- У всех вошедших в меню есть «Занятия»; у менеджера и мастера — также «Новое занятие».
- Ученик видит только занятия тех классов, в которых он состоит; учитель — только свои занятия; менеджер и мастер — все занятия.
- Создание занятия (класс, по желанию учитель, название, описание, дата, длительность, аудитория) доступно менеджеру и мастеру.
- Открытие занятия показывает данные занятия, пустой список домашних заданий и ссылку «Добавить домашнее задание» (до Части I по ней будет 404).
- Для пользователя с правом управлять занятием (учитель этого занятия, менеджер или мастер) на странице занятия отображается блок посещаемости: список учеников класса с выбором «присутствовал»/«отсутствовал» и необязательной пометкой; после отправки формы данные сохраняются и отображаются при следующем открытии.

---

## H1. Таблицы `lessons` и `attendance` в `db.py`

### Шаг H1.1. Таблица `lessons`

Откройте `db.py`, найдите функцию `init_db()`. Сразу **после** блока `CREATE TABLE IF NOT EXISTS class_students (...)` добавьте создание таблицы `lessons`:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS lessons (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            description TEXT,
            teacher_id INTEGER REFERENCES users(id) ON DELETE SET NULL,
            class_id INTEGER NOT NULL REFERENCES classes(id) ON DELETE CASCADE,
            scheduled_date TEXT NOT NULL,
            duration INTEGER NOT NULL,
            classroom TEXT,
            created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ', 'now'))
        )
    """)
```

Поля: название, описание (необязательно), учитель (необязательно), класс, дата проведения, длительность в минутах, аудитория (необязательно).

### Шаг H1.2. Таблица `attendance`

Сразу после создания таблицы `lessons` добавьте таблицу посещаемости:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS attendance (
            student_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
            lesson_id INTEGER NOT NULL REFERENCES lessons(id) ON DELETE CASCADE,
            status TEXT NOT NULL CHECK (status IN ('present', 'absent')),
            mark TEXT,
            PRIMARY KEY (student_id, lesson_id)
        )
    """)
```

Одна запись на пару (ученик, занятие): статус «присутствовал» или «отсутствовал», пометка по желанию.

---

## H2. Хелперы `is_teacher` и `can_manage_lesson`

### Шаг H2.1. Функция `is_teacher()` в `auth_utils.py`

Откройте `auth_utils.py`. Рядом с `is_master()` добавьте:

```python
def is_teacher():
    u = current_user()
    return u is not None and u["role"] == "teacher"
```

### Шаг H2.2. Передача `is_teacher` в шаблоны

В `app.py` в импорт из `auth_utils` добавьте `is_teacher` и в словарь `inject()` добавьте `"is_teacher": is_teacher`.

### Шаг H2.3. Функция `can_manage_lesson(lesson)` в `views_lessons.py`

Право управлять занятием (редактировать, выставлять посещаемость) есть у менеджера, мастера и у учителя, назначенного на это занятие. Функцию `can_manage_lesson(lesson)` вы добавите в модуль `views_lessons.py` (шаг H4.1): она принимает строку занятия (с полем `teacher_id`) и возвращает `True`, если текущий пользователь — менеджер, мастер или учитель этого занятия. Если у занятия нет учителя (`teacher_id` пустой), управлять могут только менеджер и мастер.

---

## H3. Модуль `views_lessons.py`

Создайте файл `views_lessons.py` в той же папке, где лежат `app.py` и `views_classes.py`.

### Шаг H3.1. Импорты и хелпер `can_manage_lesson`

В начало файла вставьте:

```python
from flask import render_template, redirect, url_for, request, abort, session
from db import get_conn
from auth_utils import is_logged_in, current_user, is_teacher, is_manager, is_master


def can_manage_lesson(lesson):
    """Может ли текущий пользователь управлять занятием (посещаемость и т.д.)."""
    if not is_logged_in():
        return False
    u = current_user()
    if u is None:
        return False
    if is_manager() or is_master():
        return True
    if is_teacher() and lesson["teacher_id"] == u["id"]:
        return True
    return False
```

### Шаг H3.2. Список занятий `lesson_list_view()`

Список зависит от роли: ученик — занятия своих классов; учитель — занятия, где он указан учителем; менеджер и мастер — все занятия.

```python
def lesson_list_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    u = current_user()
    if u is None:
        session.clear()
        return redirect(url_for("login_form"))

    conn = get_conn()
    if u["role"] == "student":
        lessons = conn.execute(
            """
            SELECT l.id, l.title, l.scheduled_date, l.class_id, l.teacher_id,
                   c.name AS class_name
            FROM lessons l
            JOIN classes c ON c.id = l.class_id
            WHERE l.class_id IN (SELECT class_id FROM class_students WHERE student_id = ?)
            ORDER BY l.scheduled_date DESC
            """,
            (u["id"],),
        ).fetchall()
    elif u["role"] == "teacher":
        lessons = conn.execute(
            """
            SELECT l.id, l.title, l.scheduled_date, l.class_id, l.teacher_id,
                   c.name AS class_name
            FROM lessons l
            JOIN classes c ON c.id = l.class_id
            WHERE l.teacher_id = ?
            ORDER BY l.scheduled_date DESC
            """,
            (u["id"],),
        ).fetchall()
    elif is_manager() or is_master():
        lessons = conn.execute(
            """
            SELECT l.id, l.title, l.scheduled_date, l.class_id, l.teacher_id,
                   c.name AS class_name
            FROM lessons l
            JOIN classes c ON c.id = l.class_id
            ORDER BY l.scheduled_date DESC
            """
        ).fetchall()
    else:
        lessons = []
    conn.close()

    return render_template("lesson_list.html", lessons=lessons)
```

### Шаг H3.3. Создание занятия `lesson_new_view()` и `lesson_create_view()`

Создавать занятия могут только менеджер и мастер.

```python
def lesson_new_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_manager() and not is_master():
        abort(403)

    conn = get_conn()
    classes = conn.execute("SELECT id, name FROM classes ORDER BY name").fetchall()
    teachers = conn.execute(
        "SELECT id, username FROM users WHERE role = 'teacher' AND archived_at IS NULL ORDER BY username"
    ).fetchall()
    conn.close()
    return render_template("lesson_new.html", classes=classes, teachers=teachers, error=None)


def lesson_create_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not is_manager() and not is_master():
        abort(403)

    title = (request.form.get("title") or "").strip()
    description = (request.form.get("description") or "").strip()
    class_id = request.form.get("class_id", type=int)
    teacher_id = request.form.get("teacher_id", type=int)
    scheduled_date = (request.form.get("scheduled_date") or "").strip()
    duration = request.form.get("duration", type=int)
    classroom = (request.form.get("classroom") or "").strip()

    if not title or class_id is None:
        conn = get_conn()
        classes = conn.execute("SELECT id, name FROM classes ORDER BY name").fetchall()
        teachers = conn.execute(
            "SELECT id, username FROM users WHERE role = 'teacher' AND archived_at IS NULL ORDER BY username"
        ).fetchall()
        conn.close()
        return render_template(
            "lesson_new.html",
            classes=classes,
            teachers=teachers,
            error="Укажите название и класс.",
        )

    if duration is None or duration < 1:
        duration = 45

    conn = get_conn()
    conn.execute(
        """INSERT INTO lessons (title, description, teacher_id, class_id, scheduled_date, duration, classroom)
           VALUES (?, ?, ?, ?, ?, ?, ?)""",
        (title, description, teacher_id if teacher_id else None, class_id, scheduled_date, duration, classroom or None),
    )
    conn.commit()
    conn.close()
    return redirect(url_for("lesson_list"))
```

### Шаг H3.4. Просмотр занятия `lesson_show_view(lesson_id)`

Загрузить занятие, название класса, имя учителя, список домашних заданий (пока пустой), признак `can_manage` и для блока посещаемости — список учеников класса и текущая посещаемость. Доступ к просмотру: ученик — только занятия своих классов; учитель — только свои занятия; менеджер и мастер — все. При отсутствии права — `abort(403)`.

```python
def lesson_show_view(lesson_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    u = current_user()
    if u is None:
        session.clear()
        return redirect(url_for("login_form"))

    conn = get_conn()
    lesson = conn.execute(
        """SELECT l.id, l.title, l.description, l.teacher_id, l.class_id, l.scheduled_date, l.duration, l.classroom,
                  c.name AS class_name
           FROM lessons l
           JOIN classes c ON c.id = l.class_id
           WHERE l.id = ?""",
        (lesson_id,),
    ).fetchone()
    if lesson is None:
        conn.close()
        abort(404)

    if u["role"] == "student":
        in_class = conn.execute(
            "SELECT 1 FROM class_students WHERE class_id = ? AND student_id = ?",
            (lesson["class_id"], u["id"]),
        ).fetchone()
        if not in_class:
            conn.close()
            abort(403)
    elif u["role"] == "teacher":
        if lesson["teacher_id"] != u["id"]:
            conn.close()
            abort(403)
    elif not is_manager() and not is_master():
        conn.close()
        abort(403)

    teacher_name = None
    if lesson["teacher_id"]:
        row = conn.execute("SELECT username FROM users WHERE id = ?", (lesson["teacher_id"],)).fetchone()
        teacher_name = row["username"] if row else None

    homework_list = []

    can_manage = can_manage_lesson(lesson)
    students_attendance = []
    if can_manage:
        class_students = conn.execute(
            """SELECT u.id, u.username
               FROM users u
               INNER JOIN class_students cs ON cs.student_id = u.id
               WHERE cs.class_id = ? AND u.archived_at IS NULL
               ORDER BY u.username""",
            (lesson["class_id"],),
        ).fetchall()
        for s in class_students:
            att = conn.execute(
                "SELECT status, mark FROM attendance WHERE lesson_id = ? AND student_id = ?",
                (lesson_id, s["id"]),
            ).fetchone()
            students_attendance.append({
                "id": s["id"],
                "username": s["username"],
                "status": att["status"] if att else "present",
                "mark": att["mark"] if att else "",
            })
    conn.close()

    return render_template(
        "lesson_show.html",
        lesson=lesson,
        teacher_name=teacher_name,
        homework_list=homework_list,
        can_manage=can_manage,
        students_attendance=students_attendance,
    )
```

### Шаг H3.5. Сохранение посещаемости `lesson_attendance_save_view(lesson_id)`

Принимает POST: для каждого ученика класса в форме передаются `status` и по желанию `mark`. Один запрос на (student_id, lesson_id) — вставка или замена.

```python
def lesson_attendance_save_view(lesson_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    conn = get_conn()
    lesson = conn.execute("SELECT id, class_id, teacher_id FROM lessons WHERE id = ?", (lesson_id,)).fetchone()
    if lesson is None:
        conn.close()
        abort(404)
    if not can_manage_lesson(lesson):
        conn.close()
        abort(403)

    class_students = conn.execute(
        "SELECT student_id FROM class_students WHERE class_id = ?",
        (lesson["class_id"],),
    ).fetchall()
    for row in class_students:
        sid = row["student_id"]
        status = request.form.get(f"status_{sid}") or "present"
        if status not in ("present", "absent"):
            status = "present"
        mark = (request.form.get(f"mark_{sid}") or "").strip() or None
        conn.execute(
            "INSERT OR REPLACE INTO attendance (student_id, lesson_id, status, mark) VALUES (?, ?, ?, ?)",
            (sid, lesson_id, status, mark),
        )
    conn.commit()
    conn.close()
    return redirect(url_for("lesson_show", lesson_id=lesson_id))
```

---

## H4. Маршруты занятий в `app.py`

### Шаг H4.1. Импорт

В `app.py` добавьте импорт из `views_lessons`:

```python
from views_lessons import (
    lesson_list_view,
    lesson_new_view,
    lesson_create_view,
    lesson_show_view,
    lesson_attendance_save_view,
)
```

И в импорт из `auth_utils` — `is_teacher`, а в `inject()` — `"is_teacher": is_teacher` (если ещё не добавлено в H2.2).

### Шаг H4.2. Объявление маршрутов

Добавьте маршруты (например, после маршрутов классов). Важно: маршрут с путём `/lessons/new` должен быть объявлен **выше** маршрута `/lessons/<int:lesson_id>`, иначе «new» будет воспринят как идентификатор занятия.

```python
@app.get("/lessons")
def lesson_list():
    return lesson_list_view()


@app.get("/lessons/new")
def lesson_new():
    return lesson_new_view()


@app.post("/lessons/new")
def lesson_create():
    return lesson_create_view()


@app.get("/lessons/<int:lesson_id>")
def lesson_show(lesson_id):
    return lesson_show_view(lesson_id)


@app.post("/lessons/<int:lesson_id>/attendance")
def lesson_attendance_save(lesson_id):
    return lesson_attendance_save_view(lesson_id)
```

---

## H5. Шаблоны занятий

### Шаг H5.1. Список занятий `templates/lesson_list.html`

Создайте файл `templates/lesson_list.html`:

```html
{% extends "base.html" %}

{% block title %}Занятия · Мини‑ЛМС{% endblock %}

{% block content %}
  <section class="card">
    <h1>Занятия</h1>

    {% if is_manager() or is_master() %}
      <p><a href="{{ url_for('lesson_new') }}" class="btn btn-primary">Новое занятие</a></p>
    {% endif %}

    {% if not lessons %}
      <p class="muted">Нет занятий.</p>
    {% else %}
      <table class="table">
        <thead>
          <tr>
            <th>Дата</th>
            <th>Название</th>
            <th>Класс</th>
            <th></th>
          </tr>
        </thead>
        <tbody>
          {% for les in lessons %}
            <tr>
              <td>{{ les.scheduled_date[:10] if les.scheduled_date else '' }}</td>
              <td>{{ les.title }}</td>
              <td>{{ les.class_name }}</td>
              <td><a href="{{ url_for('lesson_show', lesson_id=les.id) }}">Открыть</a></td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    {% endif %}
  </section>
{% endblock %}
```

### Шаг H5.2. Форма создания занятия `templates/lesson_new.html`

Создайте файл `templates/lesson_new.html`:

```html
{% extends "base.html" %}

{% block title %}Новое занятие · Мини‑ЛМС{% endblock %}

{% block content %}
  <section class="card">
    <h1>Новое занятие</h1>

    {% if error %}
      <p class="bad">{{ error }}</p>
    {% endif %}

    <form method="post" action="{{ url_for('lesson_create') }}" class="form">
      <div class="form-group">
        <label for="title">Название *</label>
        <input type="text" id="title" name="title" required />
      </div>
      <div class="form-group">
        <label for="description">Описание</label>
        <textarea id="description" name="description" rows="3"></textarea>
      </div>
      <div class="form-group">
        <label for="class_id">Класс *</label>
        <select id="class_id" name="class_id" required>
          <option value="">— выберите —</option>
          {% for c in classes %}
            <option value="{{ c.id }}">{{ c.name }}</option>
          {% endfor %}
        </select>
      </div>
      <div class="form-group">
        <label for="teacher_id">Учитель</label>
        <select id="teacher_id" name="teacher_id">
          <option value="">— не назначен —</option>
          {% for t in teachers %}
            <option value="{{ t.id }}">{{ t.username }}</option>
          {% endfor %}
        </select>
      </div>
      <div class="form-group">
        <label for="scheduled_date">Дата проведения *</label>
        <input type="date" id="scheduled_date" name="scheduled_date" required />
      </div>
      <div class="form-group">
        <label for="duration">Длительность (мин) *</label>
        <input type="number" id="duration" name="duration" value="45" min="1" required />
      </div>
      <div class="form-group">
        <label for="classroom">Аудитория</label>
        <input type="text" id="classroom" name="classroom" />
      </div>
      <button type="submit" class="btn btn-primary">Создать занятие</button>
    </form>

    <p style="margin-top: 16px;"><a href="{{ url_for('lesson_list') }}">К списку занятий</a></p>
  </section>
{% endblock %}
```

### Шаг H5.3. Страница занятия `templates/lesson_show.html`

Создайте файл `templates/lesson_show.html`:

```html
{% extends "base.html" %}

{% block title %}{{ lesson.title }} · Мини‑ЛМС{% endblock %}

{% block content %}
  <section class="card">
    <h1>{{ lesson.title }}</h1>
    <p><a href="{{ url_for('lesson_list') }}">← К списку занятий</a></p>

    <p><strong>Класс:</strong> {{ lesson.class_name }}</p>
    <p><strong>Дата:</strong> {{ lesson.scheduled_date[:10] if lesson.scheduled_date else '' }}</p>
    <p><strong>Длительность:</strong> {{ lesson.duration }} мин</p>
    {% if lesson.classroom %}
      <p><strong>Аудитория:</strong> {{ lesson.classroom }}</p>
    {% endif %}
    <p><strong>Учитель:</strong> {{ teacher_name or '—' }}</p>
    {% if lesson.description %}
      <p><strong>Описание:</strong><br />{{ lesson.description }}</p>
    {% endif %}

    <h2>Домашние задания</h2>
    {% if not homework_list %}
      <p class="muted">Пока нет домашних заданий.</p>
      {% if can_manage %}
        <p><a href="{{ url_for('homework_new', lesson_id=lesson.id) }}">Добавить домашнее задание</a> (маршрут будет в Части I)</p>
      {% endif %}
    {% else %}
      <ul>
        {% for hw in homework_list %}
          <li><a href="{{ url_for('homework_show', homework_id=hw.id) }}">{{ hw.title }}</a></li>
        {% endfor %}
      </ul>
      {% if can_manage %}
        <p><a href="{{ url_for('homework_new', lesson_id=lesson.id) }}">Добавить домашнее задание</a></p>
      {% endif %}
    {% endif %}

    {% if can_manage and students_attendance %}
      <h2>Посещаемость</h2>
      <form method="post" action="{{ url_for('lesson_attendance_save', lesson_id=lesson.id) }}" class="form">
        <table class="table">
          <thead>
            <tr>
              <th>Ученик</th>
              <th>Статус</th>
              <th>Пометка</th>
            </tr>
          </thead>
          <tbody>
            {% for s in students_attendance %}
              <tr>
                <td>{{ s.username }}</td>
                <td>
                  <select name="status_{{ s.id }}">
                    <option value="present" {% if s.status == 'present' %}selected{% endif %}>Присутствовал</option>
                    <option value="absent" {% if s.status == 'absent' %}selected{% endif %}>Отсутствовал</option>
                  </select>
                </td>
                <td><input type="text" name="mark_{{ s.id }}" value="{{ s.mark or '' }}" placeholder="по желанию" /></td>
              </tr>
            {% endfor %}
          </tbody>
        </table>
        <button type="submit" class="btn btn-primary">Сохранить посещаемость</button>
      </form>
    {% elif can_manage %}
      <h2>Посещаемость</h2>
      <p class="muted">В классе нет учеников.</p>
    {% endif %}
  </section>
{% endblock %}
```

Ссылка `url_for('homework_new', lesson_id=lesson.id)` будет работать после добавления соответствующего маршрута в Части I; до этого переход по ней может вести на 404.

---

## H6. Навигация в `base.html`

Добавьте в блок навигации для вошедших пользователей ссылку «Занятия» (все) и для менеджера и мастера — «Новое занятие». Разместите их, например, после «Дашборд» и перед «Классы»:

```html
<a href="{{ url_for('lesson_list') }}">Занятия</a>
{% if is_manager() or is_master() %}
  <a href="{{ url_for('lesson_new') }}">Новое занятие</a>
{% endif %}
```

---

## Итог Части H

После выполнения части в проекте есть:

- **`db.py`**: таблицы `lessons` и `attendance` в `init_db()`.
- **`auth_utils.py`**: функция `is_teacher()`; в шаблонах доступна через context_processor.
- **`views_lessons.py`**: хелпер `can_manage_lesson(lesson)`, функции `lesson_list_view()`, `lesson_new_view()`, `lesson_create_view()`, `lesson_show_view()`, `lesson_attendance_save_view()`.
- **`app.py`**: маршруты `/lessons`, `/lessons/new` (GET и POST), `/lessons/<lesson_id>`, POST `/lessons/<lesson_id>/attendance`; в контекст шаблонов передаётся `is_teacher`.
- **Шаблоны**: `lesson_list.html`, `lesson_new.html`, `lesson_show.html`; в `base.html` — пункты «Занятия» и «Новое занятие».

Список занятий фильтруется по роли; просмотр и посещаемость доступны по правам; данные готовы для Части I (домашние задания и маршрут «Добавить домашнее задание»).
{% endraw %}
