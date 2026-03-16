{% raw %}
# Часть J: Прогресс, статистика и недавние блоки на дашборде

В Части I вы добавили домашние задания, сдачу и оценку. Теперь нужны: страница «Мой прогресс» для ученика (посещаемость и оценки), страница «Статистика» для менеджера и мастера (пять итогов) и блоки «Недавние занятия» и «Недавние домашние задания» на дашборде с ссылками по ролям.

В этой части вы добавляете:

- представление и маршрут страницы «Мой прогресс» (только для ученика): процент посещаемости, список оценок по сдачам, средняя оценка;
- представление и маршрут страницы «Статистика» (только для менеджера и мастера): пять итогов (ученики, учителя, занятия, домашние задания, сдачи);
- обновление дашборда: загрузка недавних занятий и недавних домашних заданий по роли и вывод двух блоков с ссылками;
- пункты навигации «Мой прогресс» (для ученика) и «Статистика» (для менеджера и мастера).

Архивация пользователей и ограничение роли менеджера при создании пользователя остаются на Часть K.

---

## Видимо / проверяемо

После выполнения Части J:

- Ученик в меню видит «Мой прогресс». На странице отображаются процент посещаемости (присутствовал / всего занятий по своим классам), список сданных и оценённых работ (название задания, срок сдачи, оценка) и средняя оценка.
- Менеджер и мастер в меню видят «Статистика». На странице отображаются пять чисел: учеников, учителей, занятий, домашних заданий, сдач.
- После входа на дашборде у каждой роли отображаются два блока: «Недавние занятия» и «Недавние домашние задания» со ссылками на соответствующие страницы (ученик — по своим классам, учитель — свои занятия и свои задания, менеджер и мастер — общий список по последним).

---

## J1. Страница «Мой прогресс» (только ученик)

### Шаг J1.1. Представление `progress_view()` в `views_auth.py`

Доступ только для вошедшего пользователя с ролью `student`. Иначе редирект на вход или `abort(403)`.

Откройте `views_auth.py`. Добавьте импорт `abort` из `flask`, если его ещё нет. В конец файла (перед следующим модулем или в логичном месте) добавьте функцию:

```python
def progress_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    if user is None:
        session.clear()
        return redirect(url_for("login_form"))
    if user["role"] != "student":
        abort(403)

    conn = get_conn()
    total_lessons_row = conn.execute(
        """
        SELECT COUNT(DISTINCT l.id) AS c
        FROM lessons l
        WHERE l.class_id IN (SELECT class_id FROM class_students WHERE student_id = ?)
        """,
        (user["id"],),
    ).fetchone()
    total_lessons = total_lessons_row["c"] if total_lessons_row else 0

    present_row = conn.execute(
        """
        SELECT COUNT(*) AS c FROM attendance a
        INNER JOIN lessons l ON l.id = a.lesson_id
        WHERE a.student_id = ? AND a.status = 'present'
        AND l.class_id IN (SELECT class_id FROM class_students WHERE student_id = ?)
        """,
        (user["id"], user["id"]),
    ).fetchone()
    present_count = present_row["c"] if present_row else 0

    attendance_pct = (present_count / total_lessons * 100) if total_lessons else 0

    graded = conn.execute(
        """
        SELECT h.id AS homework_id, h.title AS homework_title, h.due_date, s.grade
        FROM submissions s
        JOIN homework h ON h.id = s.homework_id
        WHERE s.student_id = ? AND s.grade IS NOT NULL
        ORDER BY h.due_date DESC
        """,
        (user["id"],),
    ).fetchall()

    avg_row = conn.execute(
        "SELECT AVG(grade) AS avg_grade FROM submissions WHERE student_id = ? AND grade IS NOT NULL",
        (user["id"],),
    ).fetchone()
    average_grade = round(avg_row["avg_grade"], 1) if avg_row and avg_row["avg_grade"] is not None else None

    conn.close()

    return render_template(
        "progress.html",
        user=user,
        attendance_pct=attendance_pct,
        total_lessons=total_lessons,
        present_count=present_count,
        graded=graded,
        average_grade=average_grade,
    )
```

Переменные для шаблона: `user`, `attendance_pct`, `total_lessons`, `present_count`, `graded` (список с полями `homework_id`, `homework_title`, `due_date`, `grade`), `average_grade`.

### Шаг J1.2. Маршрут и шаблон

В `app.py` добавьте импорт `progress_view` из `views_auth` и маршрут:

```python
@app.get("/progress")
def progress():
    return progress_view()
```

Имя маршрута — `progress` (для `url_for("progress")`).

Создайте шаблон `templates/progress.html`:

```html
{% extends "base.html" %}
{% block title %}Мой прогресс · Мини‑ЛМС{% endblock %}
{% block content %}
  <section class="card">
    <h1>Мой прогресс</h1>
    <p><strong>Посещаемость:</strong> {{ "%.0f"|format(attendance_pct) }}% (присутствовал на {{ present_count }} из {{ total_lessons }} занятий)</p>
    <h2>Оценки</h2>
    {% if not graded %}
      <p class="muted">Пока нет оценённых сдач.</p>
    {% else %}
      <ul>
        {% for g in graded %}
          <li>
            <a href="{{ url_for('homework_show', homework_id=g.homework_id) }}">{{ g.homework_title }}</a>
            — срок: {{ g.due_date[:10] if g.due_date else '' }}, оценка: {{ g.grade }}
          </li>
        {% endfor %}
      </ul>
      {% if average_grade is not none %}
        <p><strong>Средняя оценка:</strong> {{ average_grade }}</p>
      {% endif %}
    {% endif %}
  </section>
{% endblock %}
```

---

## J2. Страница «Статистика» (менеджер и мастер)

### Шаг J2.1. Представление `stats_view()` в `views_auth.py`

Доступ только для вошедшего пользователя с ролью `manager` или `admin`. Иначе редирект на вход или `abort(403)`.

В `views_auth.py` добавьте функцию:

```python
def stats_view():
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))

    user = current_user()
    if user is None:
        session.clear()
        return redirect(url_for("login_form"))
    if user["role"] != "manager" and user["role"] != "admin":
        abort(403)

    conn = get_conn()
    count_students = conn.execute(
        "SELECT COUNT(*) AS c FROM users WHERE role = 'student' AND archived_at IS NULL"
    ).fetchone()["c"]
    count_teachers = conn.execute(
        "SELECT COUNT(*) AS c FROM users WHERE role = 'teacher' AND archived_at IS NULL"
    ).fetchone()["c"]
    count_lessons = conn.execute("SELECT COUNT(*) AS c FROM lessons").fetchone()["c"]
    count_homework = conn.execute("SELECT COUNT(*) AS c FROM homework").fetchone()["c"]
    count_submissions = conn.execute("SELECT COUNT(*) AS c FROM submissions").fetchone()["c"]
    conn.close()

    return render_template(
        "stats.html",
        count_students=count_students,
        count_teachers=count_teachers,
        count_lessons=count_lessons,
        count_homework=count_homework,
        count_submissions=count_submissions,
    )
```

### Шаг J2.2. Маршрут и шаблон

В `app.py` добавьте импорт `stats_view` из `views_auth` и маршрут:

```python
@app.get("/stats")
def stats():
    return stats_view()
```

Имя маршрута — `stats`.

Создайте шаблон `templates/stats.html`:

```html
{% extends "base.html" %}
{% block title %}Статистика · Мини‑ЛМС{% endblock %}
{% block content %}
  <section class="card">
    <h1>Статистика</h1>
    <ul>
      <li>Учеников: {{ count_students }}</li>
      <li>Учителей: {{ count_teachers }}</li>
      <li>Занятий: {{ count_lessons }}</li>
      <li>Домашних заданий: {{ count_homework }}</li>
      <li>Сдач: {{ count_submissions }}</li>
    </ul>
  </section>
{% endblock %}
```

---

## J3. Дашборд: блоки «Недавние занятия» и «Недавние домашние задания»

### Шаг J3.1. Загрузка данных по роли в `dashboard_view()`

В `views_auth.py` в функции `dashboard_view()` после загрузки `total_students`, `total_teachers`, `total_managers` добавьте загрузку списков недавних занятий и недавних домашних заданий в зависимости от роли. Используйте одно соединение; перед `conn.close()` выполните запросы и сохраните результаты в переменных `recent_lessons` и `recent_homework` (списки строк из БД, например по 10 последних). Затем передайте их в `render_template("dashboard.html", ...)`.

Логика по ролям:

- **Ученик (student):** недавние занятия — занятия по классам ученика (`class_id IN (SELECT class_id FROM class_students WHERE student_id = ?)`), сортировка по `scheduled_date DESC`, предел 10. Недавние домашние задания — задания по занятиям из этих классов (например, `lesson_id IN (SELECT id FROM lessons WHERE class_id IN (...))`), сортировка по `due_date DESC` или `created_at DESC`, предел 10.
- **Учитель (teacher):** недавние занятия — занятия, где `teacher_id = user["id"]`, сортировка по `scheduled_date DESC`, предел 10. Недавние домашние задания — задания, созданные этим учителем (`created_by = user["id"]`), сортировка по `created_at DESC`, предел 10.
- **Менеджер и мастер (manager, admin):** недавние занятия — все занятия, сортировка по `scheduled_date DESC`, предел 10. Недавние домашние задания — все задания, сортировка по `created_at DESC`, предел 10.

Пример запросов для ученика:
```python
recent_lessons = conn.execute(
    """SELECT l.id, l.title, l.scheduled_date, c.name AS class_name
       FROM lessons l JOIN classes c ON c.id = l.class_id
       WHERE l.class_id IN (SELECT class_id FROM class_students WHERE student_id = ?)
       ORDER BY l.scheduled_date DESC LIMIT 10""",
    (user["id"],),
).fetchall()
recent_homework = conn.execute(
    """SELECT h.id, h.title, h.due_date
       FROM homework h
       WHERE h.lesson_id IN (
         SELECT id FROM lessons
         WHERE class_id IN (SELECT class_id FROM class_students WHERE student_id = ?)
       )
       ORDER BY h.due_date DESC LIMIT 10""",
    (user["id"],),
).fetchall()
```

Для учителя:
```python
recent_lessons = conn.execute(
    """SELECT l.id, l.title, l.scheduled_date, c.name AS class_name
       FROM lessons l JOIN classes c ON c.id = l.class_id
       WHERE l.teacher_id = ? ORDER BY l.scheduled_date DESC LIMIT 10""",
    (user["id"],),
).fetchall()
recent_homework = conn.execute(
    """SELECT id, title, due_date FROM homework WHERE created_by = ? ORDER BY created_at DESC LIMIT 10""",
    (user["id"],),
).fetchall()
```

Для менеджера и мастера:
```python
recent_lessons = conn.execute(
    """SELECT l.id, l.title, l.scheduled_date, c.name AS class_name
       FROM lessons l JOIN classes c ON c.id = l.class_id
       ORDER BY l.scheduled_date DESC LIMIT 10"""
).fetchall()
recent_homework = conn.execute(
    """SELECT id, title, due_date FROM homework ORDER BY created_at DESC LIMIT 10"""
).fetchall()
```

Для роли, отличной от student, teacher, manager, admin, передайте пустые списки: `recent_lessons = []`, `recent_homework = []`. Реализуйте ветвление по `user["role"]` и в конце вызова `render_template` добавьте аргументы `recent_lessons=recent_lessons` и `recent_homework=recent_homework`.

### Шаг J3.2. Блоки в шаблоне `templates/dashboard.html`

Откройте `templates/dashboard.html`. В каждом ролевом блоке (ученик, учитель, менеджер, мастер), где это уместно, добавьте два подблока:

1. **Недавние занятия** — заголовок и список ссылок на страницу занятия: `url_for('lesson_show', lesson_id=les.id)` с текстом (название и дата). Если список пуст — текст «Нет занятий».
2. **Недавние домашние задания** — заголовок и список ссылок на страницу задания: `url_for('homework_show', homework_id=hw.id)` с текстом (название, срок сдачи). Если список пуст — текст «Нет заданий».

Пример для блока ученика:
```html
<h3>Недавние занятия</h3>
{% if recent_lessons %}
  <ul>
    {% for les in recent_lessons %}
      <li><a href="{{ url_for('lesson_show', lesson_id=les.id) }}">{{ les.title }}</a> ({{ les.scheduled_date[:10] if les.scheduled_date else '' }})</li>
    {% endfor %}
  </ul>
{% else %}
  <p class="muted">Нет занятий.</p>
{% endif %}
<h3>Недавние домашние задания</h3>
{% if recent_homework %}
  <ul>
    {% for hw in recent_homework %}
      <li><a href="{{ url_for('homework_show', homework_id=hw.id) }}">{{ hw.title }}</a> (срок: {{ hw.due_date[:10] if hw.due_date else '' }})</li>
    {% endfor %}
  </ul>
{% else %}
  <p class="muted">Нет заданий.</p>
{% endif %}
```

Аналогичные блоки добавьте в секции учителя, менеджера и мастера (используются те же переменные `recent_lessons` и `recent_homework`, заполненные для этой роли в шаге J3.1).

---

## J4. Навигация в `base.html`

Добавьте в блок навигации для вошедших пользователей:

- Для **ученика** (роль student): ссылку «Мой прогресс» на `url_for('progress')`. Разместите её, например, после «Занятия».
- Для **менеджера и мастера**: ссылку «Статистика» на `url_for('stats')`. Разместите её, например, после «Классы».

Проверьте, что в меню присутствуют: Главная, Дашборд, Занятия, (Новое занятие — для менеджера/мастера), (Мой прогресс — для ученика), Классы (для менеджера/мастера), Статистика (для менеджера/мастера), Настройки и Пользователи (для мастера).

Пример фрагмента навигации:
```html
{% if current_user() %}
  <a href="{{ url_for('dashboard') }}">Дашборд</a>
  <a href="{{ url_for('lesson_list') }}">Занятия</a>
  {% if current_user().role == 'student' %}
    <a href="{{ url_for('progress') }}">Мой прогресс</a>
  {% endif %}
  {% if is_manager() or is_master() %}
    <a href="{{ url_for('lesson_new') }}">Новое занятие</a>
    <a href="{{ url_for('class_list') }}">Классы</a>
    <a href="{{ url_for('stats') }}">Статистика</a>
  {% endif %}
  ...
{% endif %}
```

Имена маршрутов должны совпадать с объявленными в `app.py` (`progress`, `stats`, `lesson_show`, `homework_show` и т.д.).

---

## Итог Части J

После выполнения части в проекте есть:

- **`views_auth.py`**: функции `progress_view()` и `stats_view()`; обновлённая `dashboard_view()` с загрузкой `recent_lessons` и `recent_homework` по роли и передачей их в шаблон дашборда.
- **`app.py`**: маршруты GET `/progress` и GET `/stats`, делегирующие в `progress_view` и `stats_view`; в импорте из `views_auth` добавлены `progress_view` и `stats_view`.
- **Шаблоны**: `progress.html` (посещаемость, список оценок, средняя оценка), `stats.html` (пять итогов); обновлённый `dashboard.html` с блоками «Недавние занятия» и «Недавние домашние задания» со ссылками.
- **`templates/base.html`**: пункты навигации «Мой прогресс» (для ученика) и «Статистика» (для менеджера и мастера).

Ученик видит свой прогресс; менеджер и мастер — сводную статистику; дашборд по ролям показывает недавние занятия и задания со ссылками. Часть K добавит архивацию пользователей и ограничение роли менеджера при создании пользователя.
{% endraw %}
