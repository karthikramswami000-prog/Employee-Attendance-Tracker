# Employee-Attendance-Tracker
-- =========================================
-- Employee Management and Attendance Tracker
-- =========================================

-- 1. CREATE TABLES
-- =================

CREATE TABLE departments (
    dept_id SERIAL PRIMARY KEY,
    dept_name VARCHAR(50) NOT NULL
);

CREATE TABLE roles (
    role_id SERIAL PRIMARY KEY,
    role_name VARCHAR(50) NOT NULL
);

CREATE TABLE employees (
    emp_id SERIAL PRIMARY KEY,
    emp_name VARCHAR(100) NOT NULL,
    email VARCHAR(100),
    dept_id INT REFERENCES departments(dept_id),
    role_id INT REFERENCES roles(role_id),
    join_date DATE
);

CREATE TABLE attendance (
    att_id SERIAL PRIMARY KEY,
    emp_id INT REFERENCES employees(emp_id),
    check_in TIMESTAMP,
    check_out TIMESTAMP,
    status VARCHAR(20),
    created_at TIMESTAMP
);

-- =========================
-- 2. INSERT DUMMY DATA
-- =========================

INSERT INTO departments (dept_name)
VALUES ('HR'), ('IT'), ('Finance'), ('Marketing'), ('Operations');

INSERT INTO roles (role_name)
VALUES ('Manager'), ('Engineer'), ('Analyst'), ('Clerk'), ('Supervisor');

-- Insert 100 Employees
INSERT INTO employees (emp_name, email, dept_id, role_id, join_date)
SELECT
    'Employee_' || i,
    'emp' || i || '@company.com',
    (random()*4 + 1)::INT,
    (random()*4 + 1)::INT,
    CURRENT_DATE - (random()*800)::INT
FROM generate_series(1,100) i;

-- Insert 250 Attendance Records (200+)
INSERT INTO attendance (emp_id, check_in, check_out)
SELECT
    (random()*99 + 1)::INT,
    CURRENT_DATE - (random()*30)::INT + TIME '09:00',
    CURRENT_DATE - (random()*30)::INT + TIME '17:00'
FROM generate_series(1,250);

-- =========================
-- 3. TRIGGER FOR STATUS & TIMESTAMP
-- =========================

CREATE OR REPLACE FUNCTION attendance_trigger_function()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.check_in::TIME > TIME '09:30' THEN
        NEW.status := 'Late';
    ELSE
        NEW.status := 'Present';
    END IF;

    NEW.created_at := CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER attendance_trigger
BEFORE INSERT ON attendance
FOR EACH ROW
EXECUTE FUNCTION attendance_trigger_function();

-- =========================
-- 4. FUNCTION TO CALCULATE TOTAL WORK HOURS
-- =========================

CREATE OR REPLACE FUNCTION calculate_total_work_hours(p_emp_id INT)
RETURNS INTERVAL AS $$
BEGIN
    RETURN (
        SELECT SUM(check_out - check_in)
        FROM attendance
        WHERE emp_id =
