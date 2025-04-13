# Задание 2: Работа с базой данных PostgreSQL

## Установка PostgreSQL (желательно в Docker)

Если установка производится через Docker, используйте следующие команды:

```bash
docker pull postgres
```

Для запуска контейнера:

```bash
docker run --name mmerloy_postgres -e POSTGRES_PASSWORD=password -d -p 15432:5432 postgres
```

![Созданный контейнер](img\docker-container.png)


---

## Создание базы данных

```sql
create database academy;
```

---

## Создание таблиц по схеме (см. рисунок 1)

```sql
create table Students (
    s_id SERIAL PRIMARY key not null,
    name CHARACTER VARYING(30) not null,
    start_year INTEGER DEFAULT 2000 CHECK(start_year > 2000 AND start_year < 2025)
);

create table Courses (
    c_no SERIAL PRIMARY key not null,
    title CHARACTER VARYING(30) not null,
    hours INTEGER DEFAULT 1 CHECK(hours > 1)
);

create table Exams (
    s_id INTEGER not null,
    c_no INTEGER not null,
    score INTEGER DEFAULT 0,
    foreign key (s_id) REFERENCES Students (s_id) on delete restrict on update restrict,
    foreign key (c_no) REFERENCES Courses (c_no) on delete restrict on update restrict
);
```

![Схема базы данных](img\db-diagram.png)

---

## Добавление записей в таблицы

```sql
INSERT INTO Students (name, start_year) VALUES
    ('Anna Petrova', 2021),
    ('Ivan Smirnov', 2022),
    ('Maria Ivanova', 2023);

INSERT INTO Courses (title, hours) VALUES
    ('Mathematics', 40),
    ('Programming', 60),
    ('Physics', 50);

INSERT INTO Exams (s_id, c_no, score) VALUES
    (1, 1, 85), 
    (1, 2, 90),
    (2, 1, 78),
    (2, 3, 82), 
    (3, 2, 95);
```

---

## Запрос: Студенты, не сдавшие ни одного экзамена

```sql
SELECT s.s_id, s.name, s.start_year
FROM Students s
WHERE NOT EXISTS (
    SELECT 1
    FROM Exams e
    WHERE e.s_id = s.s_id
);
```

---

## Запрос: Студенты с количеством сданных экзаменов

```sql
SELECT s.s_id, s.name, COUNT(e.c_no) AS exam_count 
FROM Students s 
INNER JOIN Exams e ON s.s_id = e.s_id 
GROUP BY s.s_id, s.name;
```

---

## Запрос: Курсы со средним баллом по экзамену

```sql
SELECT c.c_no, c.title, AVG(e.score)::NUMERIC(5,2) AS average_score
FROM Courses c
LEFT JOIN Exams e ON c.c_no = e.c_no
GROUP BY c.c_no, c.title
ORDER BY average_score DESC;
```

---

## (Доп. Задание) Генерация произвольных данных с помощью Python и Faker

```python
import psycopg2
from faker import Faker
import random

fake = Faker()

db_params = {
    'dbname': 'academy',
    'user': 'postgres',
    'password': 'password',
    'host': 'localhost',
    'port': '15432'
}

NUM_STUDENTS = 20
NUM_COURSES = 10
NUM_EXAMS = 50

try:
    conn = psycopg2.connect(**db_params)
    cursor = conn.cursor()

    for _ in range(NUM_STUDENTS):
        name = fake.first_name() + ' ' + fake.last_name()
        start_year = random.randint(2001, 2024)
        cursor.execute(
            "INSERT INTO Students (name, start_year) VALUES (%s, %s)",
            (name[:30], start_year)
        )

    course_titles = [
        'Math', 'Programming', 'Physics', 'Chemistry', 'Literature',
        'History', 'Biology', 'Economics', 'Philosophy', 'English language'
    ]
    random.shuffle(course_titles)
    for i in range(min(NUM_COURSES, len(course_titles))):
        title = course_titles[i]
        hours = random.randint(2, 100)
        cursor.execute(
            "INSERT INTO Courses (title, hours) VALUES (%s, %s)",
            (title[:30], hours)
        )

    cursor.execute("SELECT s_id FROM Students")
    student_ids = [row[0] for row in cursor.fetchall()]
    cursor.execute("SELECT c_no FROM Courses")
    course_ids = [row[0] for row in cursor.fetchall()]

    for _ in range(NUM_EXAMS):
        s_id = random.choice(student_ids)
        c_no = random.choice(course_ids)
        score = random.randint(0, 100)
        cursor.execute(
            "INSERT INTO Exams (s_id, c_no, score) VALUES (%s, %s, %s)",
            (s_id, c_no, score)
        )

    conn.commit()
    print(f"Успешно добавлено {NUM_STUDENTS} студентов, {NUM_COURSES} курсов и {NUM_EXAMS} экзаменов.")

except Exception as e:
    print(f"Ошибка: {e}")
    conn.rollback()

finally:
    cursor.close()
    conn.close()
```