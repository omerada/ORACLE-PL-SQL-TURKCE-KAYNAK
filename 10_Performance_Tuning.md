# Performance Tuning ve Optimizasyon

## 1. PL/SQL Performance Fundamentals

### Performance Prensipleri

1. **Minimize Context Switches**: SQL ve PL/SQL engine arasındaki geçişleri azaltın
2. **Bulk Operations**: Tek tek işlemler yerine toplu işlemler
3. **Efficient SQL**: Optimize edilmiş SQL sorguları
4. **Memory Management**: PGA ve UGA kullanımını optimize edin
5. **Caching**: Frequently used data'yı cache'leyin

## 2. SQL Performance in PL/SQL

### Query Optimization

```sql
-- KÖTÜ: Cursor loop ile row-by-row processing
DECLARE
    CURSOR emp_cursor IS SELECT employee_id, salary FROM employees;
BEGIN
    FOR emp_rec IN emp_cursor LOOP
        UPDATE employees
        SET salary = salary * 1.1
        WHERE employee_id = emp_rec.employee_id;
    END LOOP;
    COMMIT;
END;
/

-- İYİ: Single SQL statement
BEGIN
    UPDATE employees SET salary = salary * 1.1;
    COMMIT;
END;
/

-- DAHA İYİ: Bulk operations ile
DECLARE
    TYPE emp_id_array IS TABLE OF NUMBER;
    TYPE salary_array IS TABLE OF NUMBER;

    v_emp_ids emp_id_array;
    v_salaries salary_array;

    CURSOR emp_cursor IS
        SELECT employee_id, salary * 1.1
        FROM employees;
BEGIN
    OPEN emp_cursor;
    LOOP
        FETCH emp_cursor BULK COLLECT INTO v_emp_ids, v_salaries LIMIT 1000;

        FORALL i IN 1..v_emp_ids.COUNT
            UPDATE employees
            SET salary = v_salaries(i)
            WHERE employee_id = v_emp_ids(i);

        EXIT WHEN emp_cursor%NOTFOUND;
    END LOOP;
    CLOSE emp_cursor;
    COMMIT;
END;
/
```

### Bind Variables

```sql
-- KÖTÜ: Hard-coded values
CREATE OR REPLACE FUNCTION get_emp_count_bad(p_dept_id NUMBER) RETURN NUMBER IS
    v_count NUMBER;
    v_sql VARCHAR2(200);
BEGIN
    v_sql := 'SELECT COUNT(*) FROM employees WHERE department_id = ' || p_dept_id;
    EXECUTE IMMEDIATE v_sql INTO v_count;
    RETURN v_count;
END;
/

-- İYİ: Bind variables
CREATE OR REPLACE FUNCTION get_emp_count_good(p_dept_id NUMBER) RETURN NUMBER IS
    v_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_count
    FROM employees
    WHERE department_id = p_dept_id;

    RETURN v_count;
END;
/

-- Dynamic SQL ile bind variables
CREATE OR REPLACE FUNCTION dynamic_count(p_table_name VARCHAR2, p_column VARCHAR2, p_value NUMBER)
RETURN NUMBER IS
    v_count NUMBER;
    v_sql VARCHAR2(500);
BEGIN
    v_sql := 'SELECT COUNT(*) FROM ' || p_table_name || ' WHERE ' || p_column || ' = :1';
    EXECUTE IMMEDIATE v_sql INTO v_count USING p_value;
    RETURN v_count;
END;
/
```

## 3. Bulk Operations

### BULK COLLECT

```sql
-- Temel bulk collect
DECLARE
    TYPE emp_array IS TABLE OF employees%ROWTYPE;
    v_employees emp_array;
BEGIN
    SELECT * BULK COLLECT INTO v_employees
    FROM employees
    WHERE department_id = 10;

    FOR i IN 1..v_employees.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE(v_employees(i).first_name);
    END LOOP;
END;
/

-- LIMIT ile memory management
DECLARE
    CURSOR emp_cursor IS SELECT * FROM employees;
    TYPE emp_array IS TABLE OF employees%ROWTYPE;
    v_employees emp_array;

    v_batch_size CONSTANT PLS_INTEGER := 100;
BEGIN
    OPEN emp_cursor;
    LOOP
        FETCH emp_cursor BULK COLLECT INTO v_employees LIMIT v_batch_size;

        -- Process each batch
        FOR i IN 1..v_employees.COUNT LOOP
            -- Business logic here
            NULL;
        END LOOP;

        -- Exit when no more rows
        EXIT WHEN v_employees.COUNT < v_batch_size;
    END LOOP;
    CLOSE emp_cursor;
END;
/
```

### FORALL Statement

```sql
-- Efficient bulk DML
DECLARE
    TYPE id_array IS TABLE OF NUMBER;
    TYPE name_array IS TABLE OF VARCHAR2(100);
    TYPE salary_array IS TABLE OF NUMBER;

    v_ids id_array;
    v_names name_array;
    v_salaries salary_array;
BEGIN
    -- Prepare data
    SELECT employee_id, first_name || ' ' || last_name, salary * 1.1
    BULK COLLECT INTO v_ids, v_names, v_salaries
    FROM employees
    WHERE department_id = 10;

    -- Bulk update
    FORALL i IN 1..v_ids.COUNT
        UPDATE employees
        SET salary = v_salaries(i),
            last_update_date = SYSDATE
        WHERE employee_id = v_ids(i);

    DBMS_OUTPUT.PUT_LINE('Updated ' || SQL%ROWCOUNT || ' rows');
    COMMIT;
END;
/

-- FORALL with exceptions
DECLARE
    TYPE id_array IS TABLE OF NUMBER;
    v_ids id_array := id_array(100, 101, 999, 102); -- 999 doesn't exist

    bulk_errors EXCEPTION;
    PRAGMA EXCEPTION_INIT(bulk_errors, -24381);
BEGIN
    FORALL i IN 1..v_ids.COUNT SAVE EXCEPTIONS
        UPDATE employees
        SET salary = salary * 1.1
        WHERE employee_id = v_ids(i);

EXCEPTION
    WHEN bulk_errors THEN
        DBMS_OUTPUT.PUT_LINE('Errors occurred: ' || SQL%BULK_EXCEPTIONS.COUNT);

        FOR j IN 1..SQL%BULK_EXCEPTIONS.COUNT LOOP
            DBMS_OUTPUT.PUT_LINE('Error ' || j || ': ' ||
                'Index: ' || SQL%BULK_EXCEPTIONS(j).ERROR_INDEX ||
                ', Code: ' || SQL%BULK_EXCEPTIONS(j).ERROR_CODE ||
                ', Message: ' || SQLERRM(-SQL%BULK_EXCEPTIONS(j).ERROR_CODE));
        END LOOP;
END;
/
```

## 4. Caching Strategies

### Function Result Cache

```sql
-- Result cache with deterministic function
CREATE OR REPLACE FUNCTION get_tax_rate(p_salary NUMBER)
RETURN NUMBER
DETERMINISTIC
RESULT_CACHE
IS
BEGIN
    RETURN CASE
        WHEN p_salary > 100000 THEN 0.35
        WHEN p_salary > 75000 THEN 0.30
        WHEN p_salary > 50000 THEN 0.25
        WHEN p_salary > 25000 THEN 0.20
        ELSE 0.15
    END;
END;
/

-- Package-based caching
CREATE OR REPLACE PACKAGE cache_pkg IS
    TYPE dept_cache_rec IS RECORD (
        dept_id NUMBER,
        dept_name VARCHAR2(100),
        manager_name VARCHAR2(100)
    );

    TYPE dept_cache_array IS TABLE OF dept_cache_rec INDEX BY PLS_INTEGER;

    g_dept_cache dept_cache_array;
    g_cache_loaded BOOLEAN := FALSE;

    FUNCTION get_dept_info(p_dept_id NUMBER) RETURN dept_cache_rec;
    PROCEDURE load_cache;
    PROCEDURE clear_cache;
END cache_pkg;
/

CREATE OR REPLACE PACKAGE BODY cache_pkg IS
    FUNCTION get_dept_info(p_dept_id NUMBER) RETURN dept_cache_rec IS
        v_result dept_cache_rec;
    BEGIN
        -- Load cache if not loaded
        IF NOT g_cache_loaded THEN
            load_cache;
        END IF;

        -- Try to find in cache
        IF g_dept_cache.EXISTS(p_dept_id) THEN
            RETURN g_dept_cache(p_dept_id);
        END IF;

        -- Load from database if not in cache
        SELECT d.department_id, d.department_name,
               e.first_name || ' ' || e.last_name
        INTO v_result.dept_id, v_result.dept_name, v_result.manager_name
        FROM departments d
        LEFT JOIN employees e ON d.manager_id = e.employee_id
        WHERE d.department_id = p_dept_id;

        -- Add to cache
        g_dept_cache(p_dept_id) := v_result;

        RETURN v_result;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            v_result.dept_id := p_dept_id;
            v_result.dept_name := 'Unknown';
            v_result.manager_name := 'Unknown';
            RETURN v_result;
    END;

    PROCEDURE load_cache IS
        CURSOR dept_cursor IS
            SELECT d.department_id, d.department_name,
                   e.first_name || ' ' || e.last_name as manager_name
            FROM departments d
            LEFT JOIN employees e ON d.manager_id = e.employee_id;
    BEGIN
        FOR dept_rec IN dept_cursor LOOP
            g_dept_cache(dept_rec.department_id).dept_id := dept_rec.department_id;
            g_dept_cache(dept_rec.department_id).dept_name := dept_rec.department_name;
            g_dept_cache(dept_rec.department_id).manager_name := dept_rec.manager_name;
        END LOOP;

        g_cache_loaded := TRUE;
        DBMS_OUTPUT.PUT_LINE('Cache loaded: ' || g_dept_cache.COUNT || ' departments');
    END;

    PROCEDURE clear_cache IS
    BEGIN
        g_dept_cache.DELETE;
        g_cache_loaded := FALSE;
    END;
END cache_pkg;
/
```

## 5. Memory Management

### PGA Optimization

```sql
-- Memory-efficient cursor processing
CREATE OR REPLACE PROCEDURE process_large_table IS
    CURSOR large_cursor IS
        SELECT employee_id, salary, commission_pct
        FROM employees
        ORDER BY employee_id;

    TYPE emp_array IS TABLE OF large_cursor%ROWTYPE;
    v_employees emp_array;

    -- Optimal batch size based on available memory
    v_batch_size PLS_INTEGER := 1000;
    v_processed_count NUMBER := 0;
BEGIN
    OPEN large_cursor;
    LOOP
        FETCH large_cursor BULK COLLECT INTO v_employees LIMIT v_batch_size;

        -- Process batch
        FOR i IN 1..v_employees.COUNT LOOP
            -- Memory-efficient processing
            IF v_employees(i).commission_pct IS NOT NULL THEN
                -- Business logic
                v_processed_count := v_processed_count + 1;
            END IF;
        END LOOP;

        -- Clear collection to free memory
        v_employees.DELETE;

        EXIT WHEN large_cursor%NOTFOUND;
    END LOOP;
    CLOSE large_cursor;

    DBMS_OUTPUT.PUT_LINE('Processed: ' || v_processed_count || ' records');
END;
/
```

### Avoiding Memory Leaks

```sql
-- Memory leak example (AVOID)
CREATE OR REPLACE PROCEDURE memory_leak_example IS
    TYPE big_array IS TABLE OF VARCHAR2(32767);
    v_data big_array;
BEGIN
    v_data := big_array();

    FOR i IN 1..100000 LOOP
        v_data.EXTEND;
        v_data(i) := RPAD('X', 32767, 'X'); -- Allocates lots of memory
    END LOOP;

    -- Memory not freed until procedure ends
    -- Do some work...

END; -- Memory finally freed here
/

-- Memory efficient version
CREATE OR REPLACE PROCEDURE memory_efficient_example IS
    TYPE big_array IS TABLE OF VARCHAR2(32767);
    v_data big_array;
    v_batch_size CONSTANT PLS_INTEGER := 1000;
BEGIN
    FOR batch_num IN 1..100 LOOP
        v_data := big_array(); -- Fresh collection each batch
        v_data.EXTEND(v_batch_size);

        FOR i IN 1..v_batch_size LOOP
            v_data(i) := RPAD('X', 32767, 'X');
            -- Process immediately
        END LOOP;

        -- Collection goes out of scope and memory is freed
    END LOOP;
END;
/
```

## 6. SQL Tuning in PL/SQL

### Execution Plan Analysis

```sql
-- PL/SQL block with SQL monitoring
DECLARE
    v_sql_id VARCHAR2(13);
BEGIN
    -- Enable SQL monitoring for this session
    DBMS_MONITOR.SESSION_TRACE_ENABLE(session_id => SYS_CONTEXT('USERENV','SID'),
                                      serial_num => SYS_CONTEXT('USERENV','SESSIONID'),
                                      waits => TRUE,
                                      binds => TRUE);

    -- Your PL/SQL code here
    FOR dept_rec IN (SELECT department_id, department_name FROM departments) LOOP
        FOR emp_rec IN (SELECT first_name, last_name, salary
                       FROM employees
                       WHERE department_id = dept_rec.department_id) LOOP
            -- Process employee
            NULL;
        END LOOP;
    END LOOP;

    DBMS_MONITOR.SESSION_TRACE_DISABLE(session_id => SYS_CONTEXT('USERENV','SID'),
                                       serial_num => SYS_CONTEXT('USERENV','SESSIONID'));
END;
/

-- Analyze SQL performance
SELECT sql_id, executions, elapsed_time, cpu_time, buffer_gets
FROM v$sql
WHERE sql_text LIKE '%employees%'
AND parsing_user_id = (SELECT user_id FROM all_users WHERE username = USER);
```

### Hint Usage in PL/SQL

```sql
CREATE OR REPLACE PROCEDURE optimized_report IS
    CURSOR dept_summary_cursor IS
        SELECT /*+ FIRST_ROWS(10) USE_INDEX(d DEPT_NAME_IDX) */
               d.department_name,
               COUNT(e.employee_id) as emp_count,
               AVG(e.salary) as avg_salary
        FROM departments d
        LEFT JOIN employees e ON d.department_id = e.department_id
        GROUP BY d.department_name
        ORDER BY emp_count DESC;
BEGIN
    FOR dept_rec IN dept_summary_cursor LOOP
        DBMS_OUTPUT.PUT_LINE(dept_rec.department_name || ': ' || dept_rec.emp_count);
    END LOOP;
END;
/
```

## 7. Advanced Optimization Techniques

### Pipelined Table Functions

```sql
-- Type definitions
CREATE OR REPLACE TYPE emp_summary_obj AS OBJECT (
    dept_name VARCHAR2(100),
    emp_count NUMBER,
    avg_salary NUMBER,
    total_salary NUMBER
);
/

CREATE OR REPLACE TYPE emp_summary_table AS TABLE OF emp_summary_obj;
/

-- Pipelined function for streaming results
CREATE OR REPLACE FUNCTION get_dept_summary_pipelined
RETURN emp_summary_table PIPELINED IS

    CURSOR dept_cursor IS
        SELECT d.department_name,
               COUNT(e.employee_id) as emp_count,
               AVG(NVL(e.salary, 0)) as avg_salary,
               SUM(NVL(e.salary, 0)) as total_salary
        FROM departments d
        LEFT JOIN employees e ON d.department_id = e.department_id
        GROUP BY d.department_name;

    v_summary emp_summary_obj;
BEGIN
    FOR dept_rec IN dept_cursor LOOP
        v_summary := emp_summary_obj(
            dept_rec.department_name,
            dept_rec.emp_count,
            dept_rec.avg_salary,
            dept_rec.total_salary
        );

        PIPE ROW(v_summary);
    END LOOP;

    RETURN;
END;
/

-- Usage - can be used in SQL
SELECT * FROM TABLE(get_dept_summary_pipelined());
```

### Parallel Execution

```sql
-- Enable parallel execution for PL/SQL
CREATE OR REPLACE PROCEDURE parallel_processing IS
    TYPE emp_id_array IS TABLE OF NUMBER;
    v_emp_ids emp_id_array;
BEGIN
    -- Collect IDs
    SELECT employee_id BULK COLLECT INTO v_emp_ids
    FROM employees;

    -- Process in parallel batches
    FORALL i IN 1..v_emp_ids.COUNT
        UPDATE /*+ PARALLEL(employees, 4) */ employees
        SET salary = salary * 1.1,
            last_update_date = SYSDATE
        WHERE employee_id = v_emp_ids(i);

    COMMIT;
END;
/
```

## 8. Performance Monitoring

### Custom Performance Logger

```sql
CREATE OR REPLACE PACKAGE perf_monitor IS
    TYPE timing_rec IS RECORD (
        operation_name VARCHAR2(100),
        start_time TIMESTAMP,
        end_time TIMESTAMP,
        elapsed_seconds NUMBER
    );

    TYPE timing_array IS TABLE OF timing_rec;
    g_timings timing_array := timing_array();

    PROCEDURE start_timer(p_operation VARCHAR2);
    PROCEDURE end_timer(p_operation VARCHAR2);
    PROCEDURE print_timings;
    PROCEDURE clear_timings;
END perf_monitor;
/

CREATE OR REPLACE PACKAGE BODY perf_monitor IS
    PROCEDURE start_timer(p_operation VARCHAR2) IS
        v_timing timing_rec;
    BEGIN
        v_timing.operation_name := p_operation;
        v_timing.start_time := SYSTIMESTAMP;

        g_timings.EXTEND;
        g_timings(g_timings.COUNT) := v_timing;
    END;

    PROCEDURE end_timer(p_operation VARCHAR2) IS
    BEGIN
        FOR i IN REVERSE 1..g_timings.COUNT LOOP
            IF g_timings(i).operation_name = p_operation AND
               g_timings(i).end_time IS NULL THEN
                g_timings(i).end_time := SYSTIMESTAMP;
                g_timings(i).elapsed_seconds :=
                    EXTRACT(SECOND FROM (g_timings(i).end_time - g_timings(i).start_time));
                EXIT;
            END IF;
        END LOOP;
    END;

    PROCEDURE print_timings IS
    BEGIN
        DBMS_OUTPUT.PUT_LINE('=== Performance Report ===');
        FOR i IN 1..g_timings.COUNT LOOP
            DBMS_OUTPUT.PUT_LINE(
                g_timings(i).operation_name || ': ' ||
                g_timings(i).elapsed_seconds || ' seconds'
            );
        END LOOP;
    END;

    PROCEDURE clear_timings IS
    BEGIN
        g_timings.DELETE;
    END;
END perf_monitor;
/

-- Usage example
BEGIN
    perf_monitor.start_timer('BULK_UPDATE');

    -- Your code here
    UPDATE employees SET salary = salary * 1.1;

    perf_monitor.end_timer('BULK_UPDATE');
    perf_monitor.print_timings;
END;
/
```

## 9. Database Optimization

### Statistics Management

```sql
-- Gather table statistics
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname => 'HR',
        tabname => 'EMPLOYEES',
        estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
        method_opt => 'FOR ALL COLUMNS SIZE AUTO',
        cascade => TRUE -- Include indexes
    );
END;
/

-- Gather schema statistics
BEGIN
    DBMS_STATS.GATHER_SCHEMA_STATS(
        ownname => 'HR',
        estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
        cascade => TRUE,
        degree => 4 -- Parallel degree
    );
END;
/
```

### SQL Plan Management

```sql
-- Create SQL Plan Baseline
DECLARE
    v_plans PLS_INTEGER;
BEGIN
    v_plans := DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE(
        sql_id => 'your_sql_id_here',
        plan_hash_value => your_plan_hash_value,
        enabled => 'YES',
        fixed => 'NO'
    );

    DBMS_OUTPUT.PUT_LINE('Loaded ' || v_plans || ' plans');
END;
/
```

## Performance Best Practices

### Do's

1. **Use BULK COLLECT and FORALL** for array processing
2. **Minimize context switches** between SQL and PL/SQL engines
3. **Use LIMIT clause** with BULK COLLECT for large datasets
4. **Cache frequently used data** in package variables
5. **Use bind variables** instead of literals
6. **Analyze execution plans** regularly
7. **Use pipelined functions** for large result sets

### Don'ts

1. **Avoid row-by-row processing** in loops
2. **Don't use SELECT \* ** unless necessary
3. **Avoid excessive exception handling** in loops
4. **Don't ignore memory management** with large collections
5. **Avoid functions in WHERE clauses** unless function-based indexed
6. **Don't use WHEN OTHERS** without specific handling

### Performance Checklist

- [ ] SQL statements optimized
- [ ] Appropriate indexes exist
- [ ] Statistics are current
- [ ] Bulk operations used where possible
- [ ] Memory usage optimized
- [ ] Execution plans reviewed
- [ ] Performance metrics collected

**Sonraki Bölümde:** Oracle APEX'e giriş yapacağız.
