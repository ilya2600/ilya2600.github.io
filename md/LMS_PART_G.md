{% raw %}
# Часть G: Просмотр класса и привязка учеников

В Части F вы добавили ролевой дашборд, таблицу `classes`, список и создание классов. Менеджер и мастер видят список классов, но пока нельзя открыть класс и назначить в него учеников.

В этой части вы добавляете:

- таблицу `class_students` (связь «класс — ученик»);
- страницу просмотра класса по ID с двумя списками: ученики в классе (с кнопкой «Убрать») и все ученики, не входящие в класс (с кнопкой «Добавить»);
- маршруты просмотра класса и добавления/удаления ученика в классе;
- ссылку со списка классов на страницу класса.

Занятия и домашние задания остаются на следующие части. После Части G данные о привязке учеников к классам будут использоваться в Части H: ученик без классов не увидит занятий в разделе «Занятия».

---

## Видимо / проверяемо

После выполнения Части G:

- В списке классов название класса ведёт на страницу класса.
- На странице класса видны два блока: «Ученики в классе» (с кнопкой «Убрать») и «Добавить ученика» — список учеников, которых ещё нет в классе (с кнопкой «Добавить»).
- Добавление ученика в класс переносит его в список «Ученики в классе»; кнопка «Убрать» удаляет его из класса.

---

## G1. Таблица `class_students` в `db.py`

Связь «многие ко многим» между классами и учениками: один класс — много учеников, один ученик может быть в нескольких классах.

### Шаг G1.1. Добавляем таблицу `class_students`

Откройте `db.py` и найдите функцию `init_db()`. Сразу **после** блока `CREATE TABLE IF NOT EXISTS classes (...)` добавьте создание таблицы `class_students`:

```python
    conn.execute("""
        CREATE TABLE IF NOT EXISTS class_students (
            class_id INTEGER NOT NULL REFERENCES classes(id) ON DELETE CASCADE,
            student_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
            PRIMARY KEY (class_id, student_id)
        )
    """)
```

В таблице хранятся пары (класс, ученик). Один и тот же ученик не может быть дважды в одном классе благодаря `PRIMARY KEY (class_id, student_id)`. В итоге `init_db()` создаёт таблицы `settings`, `users`, `classes`, `class_students`, затем выполняет `conn.commit()` и `conn.close()`.

---

## G2. Просмотр класса и добавление/удаление учеников в `views_classes.py`

Вся логика страницы класса и операций «добавить ученика» / «убрать ученика» остаётся в модуле `views_classes.py`.

### Шаг G2.1. Импорт `flash` (по желанию)

Для сообщений после добавления или удаления ученика можно использовать `flash`. В начале `views_classes.py` добавьте `flash` в импорт из `flask`:

```python
from flask import render_template, redirect, url_for, request, abort, flash
```

Если не используете `flash`, этот шаг можно пропустить.

### Шаг G2.2. View-функция просмотра класса

В `views_classes.py` добавьте функцию (без декоратора маршрута):

```python
def class_show_view(class_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not can_manage_classes():
        abort(403)

    conn = get_conn()
    cls = conn.execute(
        "SELECT id, name, created_at FROM classes WHERE id = ?",
        (class_id,),
    ).fetchone()
    if cls is None:
        conn.close()
        abort(404)

    students_in_class = conn.execute(
        """
        SELECT u.id, u.username
        FROM users u
        INNER JOIN class_students cs ON cs.student_id = u.id
        WHERE cs.class_id = ? AND u.archived_at IS NULL
        ORDER BY u.username
        """,
        (class_id,),
    ).fetchall()

    students_not_in_class = conn.execute(
        """
        SELECT id, username
        FROM users
        WHERE role = 'student' AND archived_at IS NULL
        AND id NOT IN (SELECT student_id FROM class_students WHERE class_id = ?)
        ORDER BY username
        """,
        (class_id,),
    ).fetchall()
    conn.close()

    return render_template(
        "class_show.html",
        cls=cls,
        students_in_class=students_in_class,
        students_not_in_class=students_not_in_class,
    )
```

Логика: проверка прав, загрузка класса по ID (404 при отсутствии), два списка — ученики в классе и ученики, не входящие в класс (роль `student`, не архивированы).

### Шаг G2.3. Добавление ученика в класс

В том же файле добавьте:

```python
def class_add_student_view(class_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not can_manage_classes():
        abort(403)

    student_id = request.form.get("student_id", type=int)
    if student_id is None:
        return redirect(url_for("class_show", class_id=class_id))

    conn = get_conn()
    cls = conn.execute("SELECT id FROM classes WHERE id = ?", (class_id,)).fetchone()
    student = conn.execute(
        "SELECT id FROM users WHERE id = ? AND role = 'student' AND archived_at IS NULL",
        (student_id,),
    ).fetchone()
    if cls is None or student is None:
        conn.close()
        abort(404)

    try:
        conn.execute(
            "INSERT INTO class_students (class_id, student_id) VALUES (?, ?)",
            (class_id, student_id),
        )
        conn.commit()
    except sqlite3.IntegrityError:
        pass
    conn.close()
    return redirect(url_for("class_show", class_id=class_id))
```

Проверяется существование класса и ученика (роль student, не архивирован). Вставка в `class_students`; при дубликате пары (IntegrityError) просто редирект обратно.

Добавьте в начало файла импорт `sqlite3` (если его ещё нет):

```python
import sqlite3
```

### Шаг G2.4. Удаление ученика из класса

В том же файле добавьте:

```python
def class_remove_student_view(class_id):
    if not is_logged_in():
        return redirect(url_for("login_form", next=request.url))
    if not can_manage_classes():
        abort(403)

    student_id = request.form.get("student_id", type=int)
    if student_id is None:
        return redirect(url_for("class_show", class_id=class_id))

    conn = get_conn()
    conn.execute(
        "DELETE FROM class_students WHERE class_id = ? AND student_id = ?",
        (class_id, student_id),
    )
    conn.commit()
    conn.close()
    return redirect(url_for("class_show", class_id=class_id))
```

---

## G3. Маршруты класса в `app.py`

Маршруты только вызывают соответствующие view-функции из `views_classes`.

### Шаг G3.1. Импорт view-функций

В `app.py` в блок импортов из `views_classes` добавьте:

```python
from views_classes import (
    class_list_view,
    class_new_view,
    class_create_view,
    class_show_view,
    class_add_student_view,
    class_remove_student_view,
)
```

### Шаг G3.2. Объявление маршрутов

Ниже маршрута `class_list` добавьте маршруты просмотра класса и добавления/удаления ученика. Маршруты `class_new` и `class_create` (страница «Новый класс») оставьте на своих местах — не перемещайте их; иначе адрес `/classes/new` может перестать работать.

```python
@app.get("/classes/<int:class_id>")
def class_show(class_id):
    return class_show_view(class_id)


@app.post("/classes/<int:class_id>/students/add")
def class_add_student(class_id):
    return class_add_student_view(class_id)


@app.post("/classes/<int:class_id>/students/remove")
def class_remove_student(class_id):
    return class_remove_student_view(class_id)
```

Имена маршрутов: `class_show`, `class_add_student`, `class_remove_student`. Они понадобятся в шаблонах (`url_for('class_show', class_id=cls.id)` и т.п.).

---

## G4. Шаблон страницы класса `templates/class_show.html`

Создайте файл `templates/class_show.html`:

```html
{% extends "base.html" %}

{% block title %}{{ cls.name }} · Мини‑ЛМС{% endblock %}

{% block content %}
  <section class="card">
    <h1>Класс: {{ cls.name }}</h1>
    <p><a href="{{ url_for('class_list') }}">← К списку классов</a></p>

    <h2>Ученики в классе</h2>
    {% if not students_in_class %}
      <p class="muted">В классе пока нет учеников.</p>
    {% else %}
      <ul>
        {% for u in students_in_class %}
          <li>
            {{ u.username }}
            <form method="post" action="{{ url_for('class_remove_student', class_id=cls.id) }}" class="inline-form" style="display: inline;">
              <input type="hidden" name="student_id" value="{{ u.id }}" />
              <button type="submit" class="btn btn-sm">Убрать</button>
            </form>
          </li>
        {% endfor %}
      </ul>
    {% endif %}

    <h2>Добавить ученика</h2>
    {% if not students_not_in_class %}
      <p class="muted">Все ученики уже в этом классе или нет других учеников.</p>
    {% else %}
      <ul>
        {% for u in students_not_in_class %}
          <li>
            {{ u.username }}
            <form method="post" action="{{ url_for('class_add_student', class_id=cls.id) }}" class="inline-form" style="display: inline;">
              <input type="hidden" name="student_id" value="{{ u.id }}" />
              <button type="submit" class="btn btn-sm btn-primary">Добавить</button>
            </form>
          </li>
        {% endfor %}
      </ul>
    {% endif %}
  </section>
{% endblock %}
```

Шаблон ожидает переменные `cls`, `students_in_class`, `students_not_in_class`, которые передаёт `class_show_view()`.

---

## G5. Ссылка на класс из списка классов

В списке классов сделайте название класса ссылкой на страницу просмотра класса.

### Шаг G5.1. Обновление `templates/class_list.html`

Откройте `templates/class_list.html`. В таблице замените ячейку с названием класса так, чтобы название вело на страницу класса:

В строке таблицы вместо простого `{{ cls.name }}` используйте ссылку:

```html
<td><a href="{{ url_for('class_show', class_id=cls.id) }}">{{ cls.name }}</a></td>
```

То есть в блоке `<tbody>` для каждой строки:

```html
{% for cls in classes %}
  <tr>
    <td>{{ cls.id }}</td>
    <td><a href="{{ url_for('class_show', class_id=cls.id) }}">{{ cls.name }}</a></td>
    <td>{{ cls.created_at[:10] if cls.created_at else '' }}</td>
  </tr>
{% endfor %}
```

---

## Итог Части G

После выполнения этой части в проекте есть:

- **`db.py`**: таблица `class_students` (class_id, student_id) в `init_db()`.
- **`views_classes.py`**: функции `class_show_view()`, `class_add_student_view()`, `class_remove_student_view()`; при необходимости импорт `flash` и `sqlite3`.
- **`app.py`**: маршруты GET `/classes/<int:class_id>`, POST `/classes/<int:class_id>/students/add`, POST `/classes/<int:class_id>/students/remove`, делегирующие в `views_classes`.
- **Шаблоны**: `class_show.html` (просмотр класса, два списка, кнопки «Добавить» / «Убрать»), обновлённый `class_list.html` (название класса — ссылка на страницу класса).

Менеджер и мастер могут открывать класс, видеть учеников в нём и добавлять/убирать учеников. Эти данные будут использоваться в Части H для отображения занятий по классам и для учеников — только занятий своих классов.
{% endraw %}
