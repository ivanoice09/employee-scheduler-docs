# Database Schema

Some important details about this schema:

1. Most of the tables names aren't exaclty the same as the Entities' name. Database naming conventions follow snake_casing and Entities follow camelCasing. 

2. The tables are named in **plural all lowercase** and entities in **singular first letter uppercase**.

3. The **primary-key** naming is the same as the **foreign-key** naming. For example, if employee_id is primary key on table employees, as a foreign key on table shifts it should be employee_id still.

## Employee

table name: "employees"

- `employee_id` - PRIMARY KEY
- `first_name`
- `middle_name`
- `last_name`
- `birth_date`
- `tax_code`
- `address`
- `city`
- `created_at`
- `updated_at`

## Week

table name: "weeks"

- `week_id` - PRIMARY KEY
- `year`
- `week_number`
- `week_start_date`
- `status`
- `created_at`
- `updated_at`
- `chk_weeks_status` - CONSTRAINT: values must be equal to DRAFT, PUBLISH, LOCKED
- `weeks_session_year_week_unique` - UNIQUE(demo_session_id, year, week_number)
- `demo_session_id` - added later

## Shift

table name: "shifts"

- `shift_id` - PRIMARY KEY
- `actual_date`
- `starts_at`
- `ends_at`
- `employee_id` - FOREIGN KEY
- `week_id` - FOREIGN KEY
- `chk_shifts_end_after_start` - CONTRAINT: ends_at > starts_at
- `demo_session_id` - added later

## Task

table name: "tasks"

- `task_id`
- `name`

## TaskAssignment

table name: "task_assignments" - specific tasks can be assigned to an employee who have a shift. Don't confuse this concept when *assigning shifts to employees*.

- `task_assignment_id` - PRIMARY KEY
- `task_starts_at`
- `task_ends_at`
- `task_id` - FOREIGN KEY
- `shift_id` - FOREIGN KEY
- `chk_task_assignments_end_after_start` - CONTRAINT: task_ends_at > task_starts_at
- `demo_session_id` - (added later) FOREIGN KEY

## demo_sessions

the demo sessions are tracked here by saving the visitor's UUID temporarely.

- `demo_session_id`
- `created_at`
- `last_active_at`

## Indexes

- `idx_weeks_demo_session_id` on weeks
- `idx_shifts_demo_session_id` on shifts
- `idx_task_assignments_demo_session_id` on task_assignments
- `idx_demo_sessions_last_active` on demo_sessions

## Cardinality relationships

- `week` 1:N `shifts`: *1* week can be associated to *many* shifts
- `employees` 1:N `shifts`: *1* employee can have *many* shifts
- `demo_sessions` 1:N `shifts`: 1 DemoSession can have *many* shifts
- `shifts` 1:N `task_assignments`: *1* shift can have *many* task_assignments
- `tasks` 1:N `task_assignments`: *1* task can be assigned *many* times
- `demo_sessions` 1:N `shifts`: *1* DemoSession can have *many* Shift rows
- `demo_sessions` 1:N `weeks`: *1* DemoSession can have *many* Week rows
- `demo_sessions` 1:N `task_assignments`: *1* DemoSession can have *many* TaskAssignments