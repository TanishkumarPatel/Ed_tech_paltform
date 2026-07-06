# 🎓 EdTech Platform — Database Management System

**A fully normalized relational database modelling an online test-prep platform, inspired by apps like Physics Wallah**

[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Database-336791?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![SQL](https://img.shields.io/badge/SQL-DDL%20%7C%20DML-orange?style=for-the-badge&logo=databricks&logoColor=white)](https://en.wikipedia.org/wiki/SQL)
[![BCNF](https://img.shields.io/badge/Normalization-BCNF-brightgreen?style=for-the-badge)](#-normalization)
</div>

---

## 🔍 Overview

This project implements a **fully normalized relational database** for an EdTech platform inspired by test-prep coaching apps like **Physics Wallah**. It models the real-world lifecycle of an online coaching business — student and faculty onboarding, course enrollment and pricing, subject-wise lectures and study material, auto-graded tests, doubt resolution, and revenue tracking — for competitive-exam courses such as **JEE, NEET, UPSC, NDA, and CA Foundation**.

### What makes this project stand out?

| Feature | Detail |
|---|---|
| **Schema Design** | 14 normalized tables, all verified in BCNF |
| **Referential Integrity** | Foreign key constraints linking students, faculty, courses, tests, and content |
| **Real-World Modeling** | Enrollments, lecture releases, mock tests, doubt panels, and revenue — not just a toy schema |
| **Complex Relationships** | Many-to-many joins for faculty↔course and subject↔course assignments |
| **Analytics-Ready** | 15 documented queries covering rankings, engagement, and revenue |

---

## 🧱 Database Schema

The database consists of **14 tables** covering the full lifecycle of an online coaching platform.

### Core Entities

| Table | Description |
|---|---|
| `users` | Base identity shared by students and faculty (name, email, phone) |
| `students` | Student profile — stream, city, state — linked to a user |
| `faculty` | Faculty profile — qualification — linked to a user |
| `courses` | Course catalog with schedule, price, and exam type (MCQ / Descriptive) |
| `tests` | Tests scoped to a course, with duration and total marks |
| `questions` | Questions belonging to a test, with type and marks |
| `answers` | Submitted answers to questions, with a correctness flag |
| `subjects` | Subject catalog shared across courses |
| `lectures` | Lecture content released under a course |
| `study_material` | Downloadable material tied to a course and subject |
| `doubt_panel` | Student-raised doubts tied to a course, subject, and question |

### Relationship / Junction Tables

| Table | Relationship |
|---|---|
| `enrollment` | Student ↔ Course (with enrollment date) |
| `course_faculty` | Course ↔ Faculty (many-to-many teaching assignment) |
| `course_subject` | Course ↔ Subject (many-to-many curriculum mapping) |

---

## 📐 Normalization

All **14 relations** in the schema have been verified to satisfy **Boyce-Codd Normal Form (BCNF)**.

The analysis, documented table-by-table in `Relational Schema.pdf`, involved:

1. Deriving the **minimal functional dependencies (FDs)** implied by each table's primary key
2. Confirming every primary key — including composite keys like `(course_name, faculty_id)` — is a **superkey** that determines every other attribute
3. Verifying the pure junction tables (`course_faculty`, `course_subject`) carry only **trivial dependencies** on their composite keys

Because every key was already a superkey, **no BCNF decomposition was required** — the schema was designed in BCNF from the start.

📄 Full FD derivations & table-by-table proofs: [`Relational Schema.pdf`](Relational%20Schema.pdf)

---

## 💻 Tech Stack

| Tool | Purpose |
|---|---|
| **PostgreSQL** | Primary RDBMS — schema (`ONLINEEd`), constraints, queries |
| **SQL (DDL / DML)** | Schema definition, sample data, analytics queries |
| **Git & GitHub** | Version control and collaboration |

---

## 📁 Project Structure

```
Ed-Tech-Platform/
│
├── DDL Script .txt              # Full schema: 14 tables with PK/FK constraints
├── Data_insertion_queries.txt   # Sample INSERT statements for every table
├── SQL_Queries.txt              # 15 real-world analytics & reporting queries
├── ER_dia.pdf                   # Entity-Relationship diagram (source)             
├── Relational Schema.pdf        # Minimal FD set + table-by-table BCNF proofs
├── ACID_Concurrency_Case_Study.md  # Race-condition walkthrough: last-seat enrollment problem
└── README.md
```

---

## 🚀 Getting Started

### Prerequisites

- PostgreSQL installed 
- `psql` CLI or pgAdmin

### Setup Instructions

**1. Clone the repository**
```bash
git clone https://github.com/your-username/Ed-Tech-Platform.git
cd Ed-Tech-Platform
```

**2. Create the database**
```sql
CREATE DATABASE ed_tech_db;
```

**3. Run the DDL script**

This creates the `ONLINEEd` schema and all 14 tables.
```bash
psql -U postgres -d ed_tech_db -f "DDL Script .txt"
```

**4. (Optional) Load sample data**
```bash
psql -U postgres -d ed_tech_db -f "Data_insertion_queries.txt"
```

---

## 🔎 Sample Queries

A few examples of the queries implemented and documented in this project:

```sql
-- Enrollment count per course
SELECT c.name AS course_name, COUNT(e.id) AS enrollment_count
FROM courses c
LEFT JOIN enrollment e ON c.name = e.course_name
GROUP BY c.name;

-- Rank list of students for a given test
SELECT
    u.first_name || ' ' || u.last_name AS student_name,
    s.phone_no,
    SUM(q.marks) AS total_marks,
    RANK() OVER (ORDER BY SUM(q.marks) DESC) AS rank
FROM answers a
JOIN questions q ON a.question_id = q.id
JOIN students s ON a.phone = s.phone_no
JOIN users u ON s.user_id = u.id
WHERE a.is_correct = TRUE AND q.test_name = 'Physics Test 1'
GROUP BY s.phone_no, u.first_name, u.last_name
ORDER BY rank;

-- Revenue generated per course
SELECT c.name AS course_name, COUNT(e.id) * c.price AS total_revenue
FROM courses c
LEFT JOIN enrollment e ON c.name = e.course_name
GROUP BY c.name, c.price;
```

📄 Full query set with use-case notes (15 queries): [`SQL_Queries.txt`](SQL_Queries.txt)

---

## 🔮 Future Improvements

This project currently covers the schema, normalization, and query layer. Natural next steps:

- [ ] **Frontend dashboard** — a React/Vue portal on top of this schema: enrolled courses & scores for students, class performance & lecture uploads for faculty, revenue & enrollment analytics for admins
- [ ] **REST API layer** — wrap the schema in a backend (Node / Django / FastAPI) so a real client app can talk to the database instead of raw SQL
- [ ] **Seat capacity & concurrency-safe enrollment** — enforce a seat limit per course and correctly handle two students racing for the last seat at the same instant. 
- [ ] **Authentication & role-based access** — password hashing, sessions, and role checks (student / faculty / admin) instead of open table access
- [ ] **Payments table** — a real `payments` table with status (pending / success / failed) and gateway reference, instead of relying on `courses.price × enrollment count` as a revenue proxy
- [ ] **Notifications** — email/SMS reminders for upcoming lectures, test schedules, and doubt-panel replies
- [ ] **Indexing** — indexes on frequently filtered foreign keys (`course_name`, `phone_no`, `test_name`) once data volume grows past the sample dataset
- [ ] **Data validation constraints** — `CHECK` constraints for a non-negative `price`, a fixed set of `exam_type` values, and valid rating ranges

---

## 👨‍💻 Author

**Tanishkumar Patel**
Developed as a **Database Management Systems (DBMS)** course project 

---

