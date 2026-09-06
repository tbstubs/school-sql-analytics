# school-sql-analytics

A relational schema for a school's Student Information System, modeled in
PostgreSQL 18 and shipped with a populated test database. It covers four of the primary
table entities a school uses: students, enrollments, invoices, and payments. The schema below breaks down all the relationships between them.

This is a **data modeling** project. The crux of the work is in the schema
design and the constraints. A small data set was generated only to show the modeling and how it functions.

---

## What this demonstrates

- Normalized relational design across 5 tables
- Resolution of a many-to-many relationship through a junction table
- Referential integrity enforced with primary and foreign key constraints
- Multi-table `JOIN` chains that traverse the full billing path
- Aggregation, filtering, ordering, and type handling in queries

---

## Schema

```
                        students_enrollments
                       ┌──────────────────────┐
                       │ fk_student_id     FK │
                       │ fk_enrollment_id  FK │
                       └───┬──────────────┬───┘
                           │              │
              ┌────────────┘              └────────────┐
              │                                        │
    ┌─────────▼──────────┐                  ┌──────────▼─────────┐
    │      students      │                  │    enrollments     │
    ├────────────────────┤                  ├────────────────────┤
    │ student_id     PK  │                  │ enrollment_id  PK  │
    │ first_name         │                  │ enrollment_name    │
    │ last_name          │                  │ date               │
    │ grade              │                  │ time               │
    │ gpa                │                  │ location           │
    │ address_1          │                  └────────────────────┘
    │ address_2          │
    │ city               │
    │ state              │
    │ zip                │
    └─────────┬──────────┘
              │ 1
              │
              │ N
    ┌─────────▼──────────┐
    │      invoices      │
    ├────────────────────┤
    │ invoice_id     PK  │
    │ fk_student_id  FK  │
    │ invoice_total      │
    └─────────┬──────────┘
              │ 1
              │
              │ N
    ┌─────────▼──────────┐
    │      payments      │
    ├────────────────────┤
    │ payment_id     PK  │
    │ payer_first_name   │
    │ payer_last_name    │
    │ payment_method     │
    │ payment_date       │
    │ payment_amount     │
    │ fk_invoice_id  FK  │
    └────────────────────┘
```

**Relationships**

| From | To | Cardinality | Enforced by |
|---|---|---|---|
| `students` | `enrollments` | many-to-many | `students_enrollments` junction table |
| `students` | `invoices` | one-to-many | `invoices_fk_student_id_fkey` |
| `invoices` | `payments` | one-to-many | `payments_fk_invoice_id_fkey` |

**Test data**

| Table | Rows |
|---|---|
| `students` | 50 |
| `enrollments` | 5 |
| `invoices` | 50 |
| `payments` | 56 |

There are more payments than invoices because an invoice can be settled by more
than one payment. There is also a partial-payment case to show the one-to-many relationship between invoices and payments.

---

## Design decisions

**A junction table, not a foreign key, for enrollments.**
A student takes many courses and a course holds many students, so neither table
can carry a foreign key to the other without duplicating rows. `students_enrollments`
resolves it into two one-to-many relationships and keeps both parent tables clean.

**`money` is a deliberately bad choice made to emulate real-world database obstacles.**
PostgreSQL's `money` type is generally discouraged since it carries a locale-dependent
format and does not work well with arithmetic or comparison. It is used here for
`invoice_total` and `payment_amount` specifically to reproduce a pattern common in
inherited production databases. The cast this forces appears in the second query under "Joins across the billing path" below:

```sql
WHERE payment_amount::numeric > 1100
```

PostgreSQL only casts `money` to `numeric` per the documentation. There is no direct route to any other numeric
type, so a cast is unavoidable the moment a currency value has to be compared against a
number. The cast itself is exact and preserves the cents.

The documented way to reach any other type is to chain a second cast through `numeric`, and
that is where the pattern turns lossy. Chaining on to `::integer` here would round to the
nearest whole dollar, half away from zero. So `$1,100.40` would fail this filter and
`$1,100.50` would pass it. The rows dropped that way produce no error and no warning.

---

## Getting started

Requires PostgreSQL 18 or later. Install from
[postgresql.org](https://www.postgresql.org/download/).

Two dumps are included:

| File | Contents |
|---|---|
| `Test_School_DB Initialized Schema.sql` | Schema only — tables, sequences, constraints |
| `Test_School_DB - With Test Data.sql` | Schema plus the populated test data |

Restore whichever you want with `pg_restore`, or load it through pgAdmin. The
dumps were created by a local `postgres` role; supply whatever credentials your
own instance uses when prompted.

```bash
createdb Test_School_DB
pg_restore -d Test_School_DB "Test_School_DB - With Test Data.sql"
```

---

## Example queries

Grouped by what each one exercises.

### Reading a single table

```sql
-- Every column and row in the students table
SELECT * FROM students;

-- Every column and row in the enrollments table
SELECT * FROM enrollments;
```

### Filtering

```sql
-- Students in 9th grade or higher
SELECT * FROM students
WHERE grade >= 9;

-- Seniors graduating summa cum laude (GPA of 3.9 or above)
SELECT first_name, last_name, gpa FROM students
WHERE grade = 12 AND gpa >= 3.9;

-- Payments received after October 1, 2025
SELECT * FROM payments
WHERE payment_date > '2025-10-01';
```

### Projection and distinctness

```sql
-- The distinct cities students live in
SELECT DISTINCT city FROM students;

-- When and where each class meets
SELECT enrollment_name, time, location FROM enrollments;
```

### Aggregation

```sql
-- How many students live in each city
SELECT city, COUNT(city) FROM students
GROUP BY city;
```

### Joins across the billing path

```sql
-- Each student and the parent who paid their invoices,
-- traversing students -> invoices -> payments
SELECT DISTINCT
    students.first_name,
    students.last_name,
    payments.payer_first_name,
    payments.payer_last_name
FROM students
INNER JOIN invoices ON invoices.fk_student_id = students.student_id
INNER JOIN payments ON payments.fk_invoice_id = invoices.invoice_id;

-- Invoices settled by a payment over $1,100, with the payment details
SELECT * FROM invoices
INNER JOIN payments ON payments.fk_invoice_id = invoices.invoice_id
WHERE payment_amount::numeric > 1100
ORDER BY payer_first_name ASC;
```

### Joins across the junction table

```sql
-- The roster for a single course, reached through students_enrollments
SELECT
    students.first_name,
    students.last_name,
    enrollments.enrollment_name
FROM students
JOIN students_enrollments ON students_enrollments.fk_student_id = students.student_id
JOIN enrollments ON enrollments.enrollment_id = students_enrollments.fk_enrollment_id
WHERE enrollment_name = 'Computer Science'
ORDER BY last_name;
```

---

## Known limitations

The list below shows some of the known limitations of the current design.

- **`students_enrollments` has no unique constraint.** Nothing prevents the same
  `(fk_student_id, fk_enrollment_id)` pair from being inserted twice, which would
  double-count a student on a roster. A composite primary key on the two columns
  would be the fix.
- **`money` is the wrong type for currency.** In a normal database design, `numeric` would be a better type choice for money values due to its compatibility with arithmetic and comparison functions. See "Design
  decisions" for more info, but using `money` as the type forces the cast in the second query under "Joins across the billing path".
- **Payer identity is denormalized onto `payments`.** A `payers` table was outside the
  scope of this schema, so each payment stores the payer's name directly. A parent paying
  for two children is recorded as two unrelated name pairs, so there is no way to ask what
  a family owes in total.
- **No indexes beyond the primary keys.** At 50 rows the planner would choose a
  sequential scan regardless, so an index here would demonstrate nothing. It would
  matter more at volume — `payments.fk_invoice_id` and `invoices.fk_student_id` are the
  first two candidates.
- **`enrollments.date` and `enrollments.time` model a single meeting**, not a
  recurring schedule. A real SIS needs a separate meeting-times table.
- **No soft deletes or audit columns.** Nothing records when a row was created or
  by whom.

---

## What I would build next

- A composite primary key on `students_enrollments`
- A `payers` table, with `payments` referencing it
- `numeric(12,2)` for currency, with the migration written out
- A view exposing outstanding balance per student, as
  `invoice_total` less the sum of payments applied
