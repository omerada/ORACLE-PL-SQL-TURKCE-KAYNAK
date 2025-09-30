# İleri PL/SQL Konuları

## 1. Collections (Koleksiyonlar)

Collections, PL/SQL'de birden fazla değeri tek bir değişkende saklamaya yarayan veri yapılarıdır.

### Associative Arrays (Index-by Tables)

```sql
DECLARE
    TYPE emp_name_array IS TABLE OF employees.first_name%TYPE INDEX BY BINARY_INTEGER;
    TYPE salary_array IS TABLE OF NUMBER INDEX BY VARCHAR2(50);

    emp_names emp_name_array;
    dept_salaries salary_array;
BEGIN
    -- Index ile atama
    emp_names(1) := 'John';
    emp_names(2) := 'Jane';
    emp_names(100) := 'Bob';

    -- String index kullanımı
    dept_salaries('IT') := 75000;
    dept_salaries('HR') := 65000;
    dept_salaries('Finance') := 80000;

    -- Elemanları okuma
    DBMS_OUTPUT.PUT_LINE('Employee 1: ' || emp_names(1));
    DBMS_OUTPUT.PUT_LINE('IT Average Salary: ' || dept_salaries('IT'));

    -- Collection methods
    DBMS_OUTPUT.PUT_LINE('Count: ' || emp_names.COUNT);
    DBMS_OUTPUT.PUT_LINE('First Index: ' || emp_names.FIRST);
    DBMS_OUTPUT.PUT_LINE('Last Index: ' || emp_names.LAST);

    -- Tüm elemanları dolaş
    FOR i IN emp_names.FIRST..emp_names.LAST LOOP
        IF emp_names.EXISTS(i) THEN
            DBMS_OUTPUT.PUT_LINE('Index ' || i || ': ' || emp_names(i));
        END IF;
    END LOOP;
END;
/
```

### Nested Tables

```sql
DECLARE
    TYPE number_list IS TABLE OF NUMBER;
    TYPE emp_record IS RECORD (
        emp_id NUMBER,
        name VARCHAR2(100),
        salary NUMBER
    );
    TYPE emp_table IS TABLE OF emp_record;

    numbers number_list;
    employees emp_table;
BEGIN
    -- Initialize edilmeli
    numbers := number_list();
    employees := emp_table();

    -- Eleman ekleme
    numbers.EXTEND(3);
    numbers(1) := 10;
    numbers(2) := 20;
    numbers(3) := 30;

    -- Record ile kullanım
    employees.EXTEND;
    employees(1).emp_id := 100;
    employees(1).name := 'John Doe';
    employees(1).salary := 5000;

    -- Bulk operations
    SELECT employee_id, first_name || ' ' || last_name, salary
    BULK COLLECT INTO employees
    FROM employees
    WHERE department_id = 10;

    -- Çok boyutlu nested table
    FOR i IN 1..employees.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE(employees(i).name || ': ' || employees(i).salary);
    END LOOP;

    -- Collection methods
    DBMS_OUTPUT.PUT_LINE('Count: ' || numbers.COUNT);
    numbers.DELETE(2); -- İkinci elemanı sil
    DBMS_OUTPUT.PUT_LINE('After delete count: ' || numbers.COUNT);

    -- Tüm elemanları sil
    numbers.DELETE;
END;
/
```

### VARRAYs (Variable-Size Arrays)

```sql
DECLARE
    TYPE phone_array IS VARRAY(3) OF VARCHAR2(15);
    TYPE score_array IS VARRAY(5) OF NUMBER(3,1);

    employee_phones phone_array;
    exam_scores score_array;
BEGIN
    -- Initialize
    employee_phones := phone_array();
    exam_scores := score_array();

    -- Eleman ekleme (EXTEND gerekli)
    employee_phones.EXTEND(2);
    employee_phones(1) := '555-1234';
    employee_phones(2) := '555-5678';

    exam_scores.EXTEND(3);
    exam_scores(1) := 85.5;
    exam_scores(2) := 92.0;
    exam_scores(3) := 78.5;

    -- VARRAY özellikleri
    DBMS_OUTPUT.PUT_LINE('Limit: ' || employee_phones.LIMIT);
    DBMS_OUTPUT.PUT_LINE('Count: ' || employee_phones.COUNT);

    -- Elemanları yazdır
    FOR i IN 1..exam_scores.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE('Score ' || i || ': ' || exam_scores(i));
    END LOOP;
END;
/
```

### Collection Operations

```sql
DECLARE
    TYPE num_array IS TABLE OF NUMBER;

    numbers1 num_array := num_array(1, 2, 3, 4, 5);
    numbers2 num_array := num_array(3, 4, 5, 6, 7);
    result_array num_array;
BEGIN
    -- MULTISET UNION (Birleşim)
    result_array := numbers1 MULTISET UNION numbers2;

    -- MULTISET INTERSECT (Kesişim)
    result_array := numbers1 MULTISET INTERSECT numbers2;

    -- MULTISET EXCEPT (Fark)
    result_array := numbers1 MULTISET EXCEPT numbers2;

    -- IS A SET (Set kontrolü)
    IF numbers1 IS A SET THEN
        DBMS_OUTPUT.PUT_LINE('numbers1 is a set');
    END IF;

    -- MEMBER OF (Üyelik kontrolü)
    IF 3 MEMBER OF numbers1 THEN
        DBMS_OUTPUT.PUT_LINE('3 is member of numbers1');
    END IF;

    -- SUBMULTISET (Alt küme kontrolü)
    IF numbers1 SUBMULTISET OF numbers2 THEN
        DBMS_OUTPUT.PUT_LINE('numbers1 is subset of numbers2');
    END IF;
END;
/
```

## 2. Dynamic SQL

### EXECUTE IMMEDIATE

```sql
CREATE OR REPLACE PROCEDURE dynamic_ddl(p_table_name VARCHAR2) IS
    v_sql VARCHAR2(4000);
BEGIN
    -- DDL operation
    v_sql := 'CREATE TABLE ' || p_table_name || ' (
                id NUMBER PRIMARY KEY,
                name VARCHAR2(100),
                created_date DATE DEFAULT SYSDATE
              )';

    EXECUTE IMMEDIATE v_sql;

    -- DML operation
    v_sql := 'INSERT INTO ' || p_table_name || ' (id, name) VALUES (1, ''Test'')';
    EXECUTE IMMEDIATE v_sql;

    COMMIT;
    DBMS_OUTPUT.PUT_LINE('Table ' || p_table_name || ' created and populated');
END;
/

-- Dynamic query with parameters
DECLARE
    v_sql VARCHAR2(1000);
    v_dept_id NUMBER := 10;
    v_min_salary NUMBER := 5000;
    v_emp_count NUMBER;
    v_avg_salary NUMBER;
BEGIN
    v_sql := 'SELECT COUNT(*), AVG(salary)
              FROM employees
              WHERE department_id = :1 AND salary > :2';

    EXECUTE IMMEDIATE v_sql
    INTO v_emp_count, v_avg_salary
    USING v_dept_id, v_min_salary;

    DBMS_OUTPUT.PUT_LINE('Count: ' || v_emp_count);
    DBMS_OUTPUT.PUT_LINE('Avg Salary: ' || v_avg_salary);
END;
/
```

### DBMS_SQL Package

```sql
CREATE OR REPLACE FUNCTION dynamic_query_dbms_sql(
    p_table_name VARCHAR2,
    p_where_clause VARCHAR2
) RETURN SYS_REFCURSOR IS
    v_cursor_id INTEGER;
    v_sql VARCHAR2(4000);
    v_result SYS_REFCURSOR;
    v_dummy INTEGER;
BEGIN
    -- Cursor açma
    v_cursor_id := DBMS_SQL.OPEN_CURSOR;

    -- SQL hazırlama
    v_sql := 'SELECT * FROM ' || p_table_name;
    IF p_where_clause IS NOT NULL THEN
        v_sql := v_sql || ' WHERE ' || p_where_clause;
    END IF;

    DBMS_SQL.PARSE(v_cursor_id, v_sql, DBMS_SQL.NATIVE);

    -- Cursor'ı REF CURSOR'a dönüştür
    v_result := DBMS_SQL.TO_REFCURSOR(v_cursor_id);

    RETURN v_result;
EXCEPTION
    WHEN OTHERS THEN
        IF DBMS_SQL.IS_OPEN(v_cursor_id) THEN
            DBMS_SQL.CLOSE_CURSOR(v_cursor_id);
        END IF;
        RAISE;
END;
/
```

### Native Dynamic SQL with Collections

```sql
DECLARE
    TYPE emp_array IS TABLE OF employees%ROWTYPE;
    v_employees emp_array;
    v_sql VARCHAR2(1000);
    v_dept_list VARCHAR2(100) := '10,20,30';
BEGIN
    v_sql := 'SELECT * FROM employees WHERE department_id IN (' || v_dept_list || ')';

    EXECUTE IMMEDIATE v_sql BULK COLLECT INTO v_employees;

    FOR i IN 1..v_employees.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE(v_employees(i).first_name || ' ' || v_employees(i).last_name);
    END LOOP;
END;
/
```

## 3. Object Types

### Object Type Definition

```sql
-- Address object type
CREATE OR REPLACE TYPE address_type AS OBJECT (
    street VARCHAR2(100),
    city VARCHAR2(50),
    postal_code VARCHAR2(10),
    country VARCHAR2(50),

    -- Member functions
    MEMBER FUNCTION get_full_address RETURN VARCHAR2,
    MEMBER PROCEDURE display_address
);
/

-- Object type body
CREATE OR REPLACE TYPE BODY address_type AS
    MEMBER FUNCTION get_full_address RETURN VARCHAR2 IS
    BEGIN
        RETURN street || ', ' || city || ', ' || postal_code || ', ' || country;
    END;

    MEMBER PROCEDURE display_address IS
    BEGIN
        DBMS_OUTPUT.PUT_LINE('Address: ' || get_full_address());
    END;
END;
/

-- Person object type
CREATE OR REPLACE TYPE person_type AS OBJECT (
    first_name VARCHAR2(50),
    last_name VARCHAR2(50),
    birth_date DATE,
    address address_type,

    -- Constructor
    CONSTRUCTOR FUNCTION person_type(
        p_first_name VARCHAR2,
        p_last_name VARCHAR2,
        p_birth_date DATE
    ) RETURN SELF AS RESULT,

    -- Member functions
    MEMBER FUNCTION get_age RETURN NUMBER,
    MEMBER FUNCTION get_full_name RETURN VARCHAR2,

    -- Map function for sorting
    MAP MEMBER FUNCTION compare_persons RETURN DATE
);
/

CREATE OR REPLACE TYPE BODY person_type AS
    CONSTRUCTOR FUNCTION person_type(
        p_first_name VARCHAR2,
        p_last_name VARCHAR2,
        p_birth_date DATE
    ) RETURN SELF AS RESULT IS
    BEGIN
        first_name := p_first_name;
        last_name := p_last_name;
        birth_date := p_birth_date;
        RETURN;
    END;

    MEMBER FUNCTION get_age RETURN NUMBER IS
    BEGIN
        RETURN TRUNC(MONTHS_BETWEEN(SYSDATE, birth_date) / 12);
    END;

    MEMBER FUNCTION get_full_name RETURN VARCHAR2 IS
    BEGIN
        RETURN first_name || ' ' || last_name;
    END;

    MAP MEMBER FUNCTION compare_persons RETURN DATE IS
    BEGIN
        RETURN birth_date;
    END;
END;
/
```

### Using Object Types

```sql
DECLARE
    v_person person_type;
    v_address address_type;
    v_person_list person_list;
BEGIN
    -- Object oluşturma
    v_address := address_type('123 Main St', 'Istanbul', '34000', 'Turkey');

    v_person := person_type('Ali', 'Veli', DATE '1990-05-15');
    v_person.address := v_address;

    -- Object methods kullanımı
    DBMS_OUTPUT.PUT_LINE('Name: ' || v_person.get_full_name());
    DBMS_OUTPUT.PUT_LINE('Age: ' || v_person.get_age());
    v_person.address.display_address();
END;
/

-- Object table
CREATE TABLE persons OF person_type;

INSERT INTO persons VALUES (
    person_type('John', 'Doe', DATE '1985-03-20')
);

-- Object queries
SELECT p.get_full_name(), p.get_age()
FROM persons p;
```

### Inheritance

```sql
-- Base type
CREATE OR REPLACE TYPE shape_type AS OBJECT (
    x NUMBER,
    y NUMBER,

    -- Abstract methods (must be overridden)
    MEMBER FUNCTION area RETURN NUMBER,
    MEMBER FUNCTION perimeter RETURN NUMBER
) NOT FINAL NOT INSTANTIABLE;
/

-- Derived type
CREATE OR REPLACE TYPE rectangle_type UNDER shape_type (
    width NUMBER,
    height NUMBER,

    -- Constructor
    CONSTRUCTOR FUNCTION rectangle_type(
        p_x NUMBER, p_y NUMBER, p_width NUMBER, p_height NUMBER
    ) RETURN SELF AS RESULT,

    -- Override methods
    OVERRIDING MEMBER FUNCTION area RETURN NUMBER,
    OVERRIDING MEMBER FUNCTION perimeter RETURN NUMBER
);
/

CREATE OR REPLACE TYPE BODY rectangle_type AS
    CONSTRUCTOR FUNCTION rectangle_type(
        p_x NUMBER, p_y NUMBER, p_width NUMBER, p_height NUMBER
    ) RETURN SELF AS RESULT IS
    BEGIN
        x := p_x;
        y := p_y;
        width := p_width;
        height := p_height;
        RETURN;
    END;

    OVERRIDING MEMBER FUNCTION area RETURN NUMBER IS
    BEGIN
        RETURN width * height;
    END;

    OVERRIDING MEMBER FUNCTION perimeter RETURN NUMBER IS
    BEGIN
        RETURN 2 * (width + height);
    END;
END;
/
```

## 4. Advanced Cursor Features

### Cursor Variables (REF CURSOR)

```sql
CREATE OR REPLACE PACKAGE cursor_pkg IS
    TYPE weak_cursor IS REF CURSOR;
    TYPE emp_cursor IS REF CURSOR RETURN employees%ROWTYPE;

    FUNCTION get_employees(p_dept_id NUMBER) RETURN weak_cursor;
    PROCEDURE process_cursor(p_cursor IN OUT weak_cursor);
END cursor_pkg;
/

CREATE OR REPLACE PACKAGE BODY cursor_pkg IS
    FUNCTION get_employees(p_dept_id NUMBER) RETURN weak_cursor IS
        v_cursor weak_cursor;
    BEGIN
        IF p_dept_id IS NOT NULL THEN
            OPEN v_cursor FOR
                SELECT * FROM employees WHERE department_id = p_dept_id;
        ELSE
            OPEN v_cursor FOR
                SELECT * FROM employees;
        END IF;

        RETURN v_cursor;
    END;

    PROCEDURE process_cursor(p_cursor IN OUT weak_cursor) IS
        v_emp employees%ROWTYPE;
    BEGIN
        LOOP
            FETCH p_cursor INTO v_emp;
            EXIT WHEN p_cursor%NOTFOUND;

            DBMS_OUTPUT.PUT_LINE(v_emp.first_name || ' ' || v_emp.last_name);
        END LOOP;

        CLOSE p_cursor;
    END;
END cursor_pkg;
/
```

### Cursor Expressions

```sql
SELECT d.department_name,
       CURSOR(SELECT e.first_name, e.last_name, e.salary
              FROM employees e
              WHERE e.department_id = d.department_id
              ORDER BY e.salary DESC) as employees
FROM departments d;
```

### Bulk Collect with LIMIT

```sql
DECLARE
    CURSOR emp_cursor IS
        SELECT employee_id, first_name, salary FROM employees;

    TYPE emp_array IS TABLE OF emp_cursor%ROWTYPE;
    v_employees emp_array;

    v_batch_size CONSTANT PLS_INTEGER := 1000;
BEGIN
    OPEN emp_cursor;
    LOOP
        FETCH emp_cursor BULK COLLECT INTO v_employees LIMIT v_batch_size;

        -- Process batch
        FOR i IN 1..v_employees.COUNT LOOP
            -- Business logic here
            DBMS_OUTPUT.PUT_LINE(v_employees(i).first_name);
        END LOOP;

        EXIT WHEN emp_cursor%NOTFOUND;
    END LOOP;
    CLOSE emp_cursor;
END;
/
```

## 5. Advanced Package Features

### Package State and Persistence

```sql
CREATE OR REPLACE PACKAGE session_pkg IS
    -- Global variables persist for session
    g_user_id NUMBER;
    g_session_start DATE := SYSDATE;
    g_call_count NUMBER := 0;

    -- Initialization section runs once per session
    PROCEDURE initialize_session(p_user_id NUMBER);
    FUNCTION get_session_info RETURN VARCHAR2;
    PROCEDURE increment_call_count;
END session_pkg;
/

CREATE OR REPLACE PACKAGE BODY session_pkg IS
    -- Private variables
    g_private_key VARCHAR2(32);

    PROCEDURE initialize_session(p_user_id NUMBER) IS
    BEGIN
        g_user_id := p_user_id;
        g_session_start := SYSDATE;
        g_call_count := 0;

        -- Generate session key
        g_private_key := DBMS_RANDOM.STRING('X', 32);

        DBMS_OUTPUT.PUT_LINE('Session initialized for user: ' || p_user_id);
    END;

    FUNCTION get_session_info RETURN VARCHAR2 IS
    BEGIN
        RETURN 'User: ' || g_user_id ||
               ', Start: ' || TO_CHAR(g_session_start, 'DD/MM/YYYY HH24:MI:SS') ||
               ', Calls: ' || g_call_count;
    END;

    PROCEDURE increment_call_count IS
    BEGIN
        g_call_count := g_call_count + 1;
    END;

BEGIN
    -- Package initialization section
    DBMS_OUTPUT.PUT_LINE('Package session_pkg initialized at: ' ||
                         TO_CHAR(SYSDATE, 'DD/MM/YYYY HH24:MI:SS'));
END session_pkg;
/
```

### Overloaded Procedures

```sql
CREATE OR REPLACE PACKAGE math_pkg IS
    -- Overloaded functions
    FUNCTION calculate_area(p_radius NUMBER) RETURN NUMBER; -- Circle
    FUNCTION calculate_area(p_width NUMBER, p_height NUMBER) RETURN NUMBER; -- Rectangle
    FUNCTION calculate_area(p_base NUMBER, p_height NUMBER, p_shape VARCHAR2) RETURN NUMBER; -- Triangle

    -- Overloaded procedures
    PROCEDURE log_message(p_message VARCHAR2);
    PROCEDURE log_message(p_message VARCHAR2, p_level VARCHAR2);
    PROCEDURE log_message(p_message VARCHAR2, p_level VARCHAR2, p_module VARCHAR2);
END math_pkg;
/

CREATE OR REPLACE PACKAGE BODY math_pkg IS
    FUNCTION calculate_area(p_radius NUMBER) RETURN NUMBER IS
    BEGIN
        RETURN 3.14159 * p_radius * p_radius;
    END;

    FUNCTION calculate_area(p_width NUMBER, p_height NUMBER) RETURN NUMBER IS
    BEGIN
        RETURN p_width * p_height;
    END;

    FUNCTION calculate_area(p_base NUMBER, p_height NUMBER, p_shape VARCHAR2) RETURN NUMBER IS
    BEGIN
        IF UPPER(p_shape) = 'TRIANGLE' THEN
            RETURN 0.5 * p_base * p_height;
        ELSE
            RAISE_APPLICATION_ERROR(-20001, 'Unsupported shape: ' || p_shape);
        END IF;
    END;

    PROCEDURE log_message(p_message VARCHAR2) IS
    BEGIN
        log_message(p_message, 'INFO');
    END;

    PROCEDURE log_message(p_message VARCHAR2, p_level VARCHAR2) IS
    BEGIN
        log_message(p_message, p_level, 'GENERAL');
    END;

    PROCEDURE log_message(p_message VARCHAR2, p_level VARCHAR2, p_module VARCHAR2) IS
    BEGIN
        INSERT INTO application_log (log_date, level, module, message)
        VALUES (SYSDATE, p_level, p_module, p_message);
    END;
END math_pkg;
/
```

## 6. Advanced Exception Handling

### User-Defined Exception Classes

```sql
CREATE OR REPLACE PACKAGE exception_pkg IS
    -- Custom exception types
    business_rule_violation EXCEPTION;
    PRAGMA EXCEPTION_INIT(business_rule_violation, -20001);

    data_not_found EXCEPTION;
    PRAGMA EXCEPTION_INIT(data_not_found, -20002);

    invalid_parameter EXCEPTION;
    PRAGMA EXCEPTION_INIT(invalid_parameter, -20003);

    -- Exception handling utilities
    PROCEDURE raise_business_error(p_message VARCHAR2);
    PROCEDURE log_and_raise(p_error_code NUMBER, p_message VARCHAR2);
    FUNCTION get_error_stack RETURN CLOB;
END exception_pkg;
/

CREATE OR REPLACE PACKAGE BODY exception_pkg IS
    PROCEDURE raise_business_error(p_message VARCHAR2) IS
    BEGIN
        RAISE_APPLICATION_ERROR(-20001, 'Business Rule Violation: ' || p_message);
    END;

    PROCEDURE log_and_raise(p_error_code NUMBER, p_message VARCHAR2) IS
    BEGIN
        -- Log the error
        INSERT INTO error_log (error_date, error_code, error_message, call_stack)
        VALUES (SYSDATE, p_error_code, p_message, DBMS_UTILITY.FORMAT_ERROR_STACK);

        COMMIT;

        -- Re-raise the error
        RAISE_APPLICATION_ERROR(p_error_code, p_message);
    END;

    FUNCTION get_error_stack RETURN CLOB IS
    BEGIN
        RETURN DBMS_UTILITY.FORMAT_ERROR_STACK || CHR(10) ||
               DBMS_UTILITY.FORMAT_ERROR_BACKTRACE;
    END;
END exception_pkg;
/
```

### Exception Propagation Chain

```sql
CREATE OR REPLACE PROCEDURE nested_exception_demo IS
    PROCEDURE level3_proc IS
    BEGIN
        -- Simulate error at level 3
        RAISE NO_DATA_FOUND;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            DBMS_OUTPUT.PUT_LINE('Level 3: Caught NO_DATA_FOUND');
            RAISE_APPLICATION_ERROR(-20003, 'Level 3 Error: No data found');
    END level3_proc;

    PROCEDURE level2_proc IS
    BEGIN
        level3_proc;
    EXCEPTION
        WHEN OTHERS THEN
            DBMS_OUTPUT.PUT_LINE('Level 2: Caught error: ' || SQLERRM);
            RAISE_APPLICATION_ERROR(-20002, 'Level 2 Error: ' || SQLERRM);
    END level2_proc;

    PROCEDURE level1_proc IS
    BEGIN
        level2_proc;
    EXCEPTION
        WHEN OTHERS THEN
            DBMS_OUTPUT.PUT_LINE('Level 1: Caught error: ' || SQLERRM);
            DBMS_OUTPUT.PUT_LINE('Error Stack:');
            DBMS_OUTPUT.PUT_LINE(DBMS_UTILITY.FORMAT_ERROR_STACK);
            DBMS_OUTPUT.PUT_LINE('Call Stack:');
            DBMS_OUTPUT.PUT_LINE(DBMS_UTILITY.FORMAT_CALL_STACK);

            -- Log error and continue
            exception_pkg.log_and_raise(-20001, 'Level 1 Error: ' || SQLERRM);
    END level1_proc;
BEGIN
    level1_proc;
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Main: Final exception handler');
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
END;
/
```

## Pratik Kullanım Örnekleri

### Audit Framework with Collections

```sql
CREATE OR REPLACE PACKAGE audit_framework IS
    TYPE change_record IS RECORD (
        table_name VARCHAR2(30),
        column_name VARCHAR2(30),
        old_value VARCHAR2(4000),
        new_value VARCHAR2(4000),
        change_type VARCHAR2(10)
    );

    TYPE change_array IS TABLE OF change_record;

    g_changes change_array := change_array();

    PROCEDURE add_change(
        p_table VARCHAR2,
        p_column VARCHAR2,
        p_old_value VARCHAR2,
        p_new_value VARCHAR2,
        p_type VARCHAR2
    );

    PROCEDURE save_audit_trail(p_transaction_id VARCHAR2);
    PROCEDURE clear_changes;
END audit_framework;
/
```

### Configuration Management

```sql
CREATE OR REPLACE PACKAGE config_mgmt IS
    TYPE config_rec IS RECORD (
        config_key VARCHAR2(100),
        config_value VARCHAR2(4000),
        data_type VARCHAR2(20),
        is_encrypted BOOLEAN
    );

    TYPE config_cache IS TABLE OF config_rec INDEX BY VARCHAR2(100);

    g_config_cache config_cache;
    g_cache_loaded BOOLEAN := FALSE;

    FUNCTION get_config(p_key VARCHAR2) RETURN VARCHAR2;
    FUNCTION get_config_number(p_key VARCHAR2) RETURN NUMBER;
    FUNCTION get_config_date(p_key VARCHAR2) RETURN DATE;
    FUNCTION get_config_boolean(p_key VARCHAR2) RETURN BOOLEAN;

    PROCEDURE set_config(p_key VARCHAR2, p_value VARCHAR2);
    PROCEDURE refresh_cache;
END config_mgmt;
/
```

**Sonraki Bölümde:** Performance Tuning ve optimizasyon tekniklerini öğreneceğiz.
