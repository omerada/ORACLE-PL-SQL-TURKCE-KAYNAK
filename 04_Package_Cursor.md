# Package ve Cursor Yapıları

## 1. Package Nedir?

Package, ilgili procedure, function, değişken ve type'ları gruplandıran mantıksal bir yapıdır. İki bölümden oluşur:

- **Specification (SPEC)**: Dışarıya açık interface
- **Body**: Gerçek implementasyon

**Neden Kullanılır?**

- Modüler programlama
- Encapsulation (kapsülleme)
- Namespace yönetimi
- Performans artışı (ilk çağrıda hafızaya yüklenir)
- Global değişkenler

## 2. Package Oluşturma

### Package Specification

```sql
CREATE OR REPLACE PACKAGE employee_mgmt IS
    -- Public constants
    c_max_salary CONSTANT NUMBER := 50000;
    c_min_salary CONSTANT NUMBER := 2000;

    -- Public variables
    g_last_action VARCHAR2(100);

    -- Public types
    TYPE emp_record IS RECORD (
        emp_id NUMBER,
        full_name VARCHAR2(100),
        salary NUMBER,
        hire_date DATE
    );

    TYPE emp_array IS TABLE OF emp_record;

    -- Public procedures and functions
    PROCEDURE add_employee(
        p_first_name VARCHAR2,
        p_last_name VARCHAR2,
        p_email VARCHAR2,
        p_job_id VARCHAR2,
        p_salary NUMBER,
        p_dept_id NUMBER
    );

    FUNCTION get_employee_count(p_dept_id NUMBER) RETURN NUMBER;

    PROCEDURE increase_salary(
        p_emp_id NUMBER,
        p_percentage NUMBER
    );

    FUNCTION validate_salary(p_salary NUMBER) RETURN BOOLEAN;

END employee_mgmt;
/
```

### Package Body

```sql
CREATE OR REPLACE PACKAGE BODY employee_mgmt IS

    -- Private variables
    g_package_name CONSTANT VARCHAR2(30) := 'EMPLOYEE_MGMT';

    -- Private function (sadece package içinde kullanılır)
    FUNCTION generate_employee_id RETURN NUMBER IS
        v_new_id NUMBER;
    BEGIN
        SELECT NVL(MAX(employee_id), 0) + 1
        INTO v_new_id
        FROM employees;

        RETURN v_new_id;
    END generate_employee_id;

    -- Public function implementation
    FUNCTION validate_salary(p_salary NUMBER) RETURN BOOLEAN IS
    BEGIN
        RETURN (p_salary BETWEEN c_min_salary AND c_max_salary);
    END validate_salary;

    -- Public procedure implementation
    PROCEDURE add_employee(
        p_first_name VARCHAR2,
        p_last_name VARCHAR2,
        p_email VARCHAR2,
        p_job_id VARCHAR2,
        p_salary NUMBER,
        p_dept_id NUMBER
    ) IS
        v_emp_id NUMBER;
    BEGIN
        -- Salary validation
        IF NOT validate_salary(p_salary) THEN
            RAISE_APPLICATION_ERROR(-20001,
                'Maaş ' || c_min_salary || ' - ' || c_max_salary || ' arasında olmalı');
        END IF;

        -- Generate new employee ID
        v_emp_id := generate_employee_id;

        -- Insert employee
        INSERT INTO employees (
            employee_id, first_name, last_name, email,
            hire_date, job_id, salary, department_id
        ) VALUES (
            v_emp_id, p_first_name, p_last_name, p_email,
            SYSDATE, p_job_id, p_salary, p_dept_id
        );

        g_last_action := 'Eklendi: ' || p_first_name || ' ' || p_last_name ||
                         ' (ID: ' || v_emp_id || ')';

        COMMIT;
    EXCEPTION
        WHEN OTHERS THEN
            ROLLBACK;
            g_last_action := 'Hata: ' || SQLERRM;
            RAISE;
    END add_employee;

    FUNCTION get_employee_count(p_dept_id NUMBER) RETURN NUMBER IS
        v_count NUMBER;
    BEGIN
        SELECT COUNT(*)
        INTO v_count
        FROM employees
        WHERE department_id = p_dept_id;

        RETURN v_count;
    END get_employee_count;

    PROCEDURE increase_salary(
        p_emp_id NUMBER,
        p_percentage NUMBER
    ) IS
        v_current_salary NUMBER;
        v_new_salary NUMBER;
        v_emp_name VARCHAR2(100);
    BEGIN
        SELECT salary, first_name || ' ' || last_name
        INTO v_current_salary, v_emp_name
        FROM employees
        WHERE employee_id = p_emp_id;

        v_new_salary := v_current_salary * (1 + p_percentage / 100);

        IF NOT validate_salary(v_new_salary) THEN
            RAISE_APPLICATION_ERROR(-20002,
                'Yeni maaş limit aşıyor: ' || v_new_salary);
        END IF;

        UPDATE employees
        SET salary = v_new_salary
        WHERE employee_id = p_emp_id;

        g_last_action := v_emp_name || ' maaşı %' || p_percentage || ' artırıldı';

        COMMIT;
    END increase_salary;

END employee_mgmt;
/
```

### Package Kullanımı

```sql
BEGIN
    -- Çalışan ekleme
    employee_mgmt.add_employee(
        p_first_name => 'Ali',
        p_last_name => 'Veli',
        p_email => 'ali.veli@company.com',
        p_job_id => 'IT_PROG',
        p_salary => 8000,
        p_dept_id => 60
    );

    DBMS_OUTPUT.PUT_LINE(employee_mgmt.g_last_action);

    -- Departman çalışan sayısı
    DBMS_OUTPUT.PUT_LINE('IT departmanı çalışan sayısı: ' ||
                         employee_mgmt.get_employee_count(60));
END;
/
```

## 3. Cursor Nedir?

Cursor, SQL sorgu sonuçlarını satır satır işlemek için kullanılan yapıdır.

**Ne Zaman Kullanılır?**

- Çok sayıda kaydı işlerken
- Her satır için farklı işlemler yapmak gerektiğinde
- Memory kullanımını kontrol etmek için

## 4. Implicit vs Explicit Cursor

### Implicit Cursor (Otomatik)

```sql
BEGIN
    UPDATE employees
    SET salary = salary * 1.1
    WHERE department_id = 10;

    DBMS_OUTPUT.PUT_LINE(SQL%ROWCOUNT || ' kayıt güncellendi');

    IF SQL%FOUND THEN
        DBMS_OUTPUT.PUT_LINE('En az bir kayıt etkilendi');
    END IF;
END;
/
```

### Explicit Cursor (Manuel)

```sql
DECLARE
    CURSOR emp_cursor IS
        SELECT employee_id, first_name, last_name, salary
        FROM employees
        WHERE department_id = 50
        ORDER BY salary DESC;

    v_emp emp_cursor%ROWTYPE;
BEGIN
    OPEN emp_cursor;
    LOOP
        FETCH emp_cursor INTO v_emp;
        EXIT WHEN emp_cursor%NOTFOUND;

        DBMS_OUTPUT.PUT_LINE(v_emp.first_name || ' ' || v_emp.last_name ||
                           ' - Maaş: ' || v_emp.salary);
    END LOOP;
    CLOSE emp_cursor;
END;
/
```

## 5. Cursor FOR Loop

En yaygın ve kolay kullanım:

```sql
DECLARE
    CURSOR dept_summary_cursor IS
        SELECT d.department_name,
               COUNT(e.employee_id) as emp_count,
               AVG(e.salary) as avg_salary,
               MAX(e.salary) as max_salary,
               MIN(e.salary) as min_salary
        FROM departments d
        LEFT JOIN employees e ON d.department_id = e.department_id
        GROUP BY d.department_id, d.department_name
        ORDER BY emp_count DESC;
BEGIN
    FOR dept_rec IN dept_summary_cursor LOOP
        DBMS_OUTPUT.PUT_LINE('=== ' || dept_rec.department_name || ' ===');
        DBMS_OUTPUT.PUT_LINE('Çalışan Sayısı: ' || dept_rec.emp_count);

        IF dept_rec.emp_count > 0 THEN
            DBMS_OUTPUT.PUT_LINE('Ortalama Maaş: ' || ROUND(dept_rec.avg_salary, 2));
            DBMS_OUTPUT.PUT_LINE('En Yüksek Maaş: ' || dept_rec.max_salary);
            DBMS_OUTPUT.PUT_LINE('En Düşük Maaş: ' || dept_rec.min_salary);
        END IF;

        DBMS_OUTPUT.PUT_LINE('');
    END LOOP;
END;
/
```

## 6. Parametreli Cursor

```sql
DECLARE
    CURSOR emp_by_salary_cursor(p_min_salary NUMBER, p_max_salary NUMBER) IS
        SELECT employee_id, first_name, last_name, salary, job_id
        FROM employees
        WHERE salary BETWEEN p_min_salary AND p_max_salary
        ORDER BY salary DESC;

    v_bonus NUMBER;
BEGIN
    FOR emp_rec IN emp_by_salary_cursor(5000, 15000) LOOP
        -- Bonus hesaplama
        v_bonus := emp_rec.salary * 0.1;

        DBMS_OUTPUT.PUT_LINE(emp_rec.first_name || ' ' || emp_rec.last_name ||
                           ' - Maaş: ' || emp_rec.salary ||
                           ' - Bonus: ' || v_bonus);
    END LOOP;
END;
/
```

## 7. Cursor with UPDATE

```sql
DECLARE
    CURSOR emp_update_cursor IS
        SELECT employee_id, salary, hire_date
        FROM employees
        WHERE department_id = 50
        FOR UPDATE; -- Row-level lock

    v_years_worked NUMBER;
    v_bonus_percent NUMBER;
BEGIN
    FOR emp_rec IN emp_update_cursor LOOP
        -- Çalışma yılı hesapla
        v_years_worked := TRUNC(MONTHS_BETWEEN(SYSDATE, emp_rec.hire_date) / 12);

        -- Kıdem bonusu hesapla
        v_bonus_percent := CASE
            WHEN v_years_worked > 10 THEN 15
            WHEN v_years_worked > 5 THEN 10
            WHEN v_years_worked > 2 THEN 5
            ELSE 0
        END;

        IF v_bonus_percent > 0 THEN
            UPDATE employees
            SET salary = salary * (1 + v_bonus_percent / 100)
            WHERE CURRENT OF emp_update_cursor;

            DBMS_OUTPUT.PUT_LINE('Çalışan ID: ' || emp_rec.employee_id ||
                               ' - %' || v_bonus_percent || ' artış');
        END IF;
    END LOOP;

    COMMIT;
END;
/
```

## 8. Bulk Collect ile Performans

```sql
DECLARE
    TYPE emp_id_array IS TABLE OF NUMBER;
    TYPE salary_array IS TABLE OF NUMBER;

    v_emp_ids emp_id_array;
    v_salaries salary_array;

    CURSOR bulk_cursor IS
        SELECT employee_id, salary
        FROM employees
        WHERE department_id = 50;
BEGIN
    OPEN bulk_cursor;
    LOOP
        FETCH bulk_cursor BULK COLLECT INTO v_emp_ids, v_salaries LIMIT 100;

        FOR i IN 1..v_emp_ids.COUNT LOOP
            DBMS_OUTPUT.PUT_LINE('ID: ' || v_emp_ids(i) ||
                               ' - Maaş: ' || v_salaries(i));
        END LOOP;

        EXIT WHEN bulk_cursor%NOTFOUND;
    END LOOP;
    CLOSE bulk_cursor;
END;
/
```

## 9. Cursor Attributes

```sql
DECLARE
    CURSOR test_cursor IS SELECT * FROM employees WHERE rownum <= 5;
    v_emp employees%ROWTYPE;
BEGIN
    OPEN test_cursor;

    DBMS_OUTPUT.PUT_LINE('Cursor açık mı? ' ||
        CASE WHEN test_cursor%ISOPEN THEN 'Evet' ELSE 'Hayır' END);

    LOOP
        FETCH test_cursor INTO v_emp;

        DBMS_OUTPUT.PUT_LINE('Toplam fetch: ' || test_cursor%ROWCOUNT);
        DBMS_OUTPUT.PUT_LINE('Veri bulundu mu? ' ||
            CASE WHEN test_cursor%FOUND THEN 'Evet' ELSE 'Hayır' END);

        EXIT WHEN test_cursor%NOTFOUND;
    END LOOP;

    CLOSE test_cursor;
END;
/
```

## 10. Gerçek Dünya Package Örneği

```sql
CREATE OR REPLACE PACKAGE reporting_pkg IS
    TYPE monthly_report_rec IS RECORD (
        month_name VARCHAR2(20),
        total_sales NUMBER,
        avg_order_value NUMBER,
        customer_count NUMBER
    );

    TYPE report_array IS TABLE OF monthly_report_rec;

    FUNCTION get_monthly_sales(p_year NUMBER) RETURN report_array PIPELINED;

    PROCEDURE generate_excel_report(
        p_year NUMBER,
        p_file_path VARCHAR2
    );

END reporting_pkg;
/

CREATE OR REPLACE PACKAGE BODY reporting_pkg IS

    FUNCTION get_monthly_sales(p_year NUMBER) RETURN report_array PIPELINED IS
        v_report monthly_report_rec;

        CURSOR monthly_cursor IS
            SELECT TO_CHAR(order_date, 'Month') as month_name,
                   SUM(total_amount) as total_sales,
                   AVG(total_amount) as avg_order,
                   COUNT(DISTINCT customer_id) as customer_count
            FROM orders
            WHERE EXTRACT(YEAR FROM order_date) = p_year
            GROUP BY TO_CHAR(order_date, 'Month'), EXTRACT(MONTH FROM order_date)
            ORDER BY EXTRACT(MONTH FROM order_date);
    BEGIN
        FOR rec IN monthly_cursor LOOP
            v_report.month_name := rec.month_name;
            v_report.total_sales := rec.total_sales;
            v_report.avg_order_value := rec.avg_order;
            v_report.customer_count := rec.customer_count;

            PIPE ROW(v_report);
        END LOOP;

        RETURN;
    END get_monthly_sales;

    PROCEDURE generate_excel_report(
        p_year NUMBER,
        p_file_path VARCHAR2
    ) IS
        -- Excel generation logic
    BEGIN
        NULL; -- Implementation would go here
    END generate_excel_report;

END reporting_pkg;
/
```

## Best Practices

1. **Package Organization**: İlgili fonksiyonları grupla
2. **Naming**: Anlamlı ve tutarlı isimlendirme
3. **Cursor Memory**: Büyük result set'ler için BULK COLLECT kullan
4. **Exception Handling**: Her cursor operation'da hata yönetimi
5. **Resource Management**: Cursor'ları mutlaka kapat

## Pratik Kullanım Alanları

1. **Data Migration**: Büyük veri taşıma işlemleri
2. **Batch Processing**: Toplu işlem routineleri
3. **Reporting**: Karmaşık rapor hesaplamaları
4. **API Development**: Modüler servis katmanları
5. **ETL Operations**: Extract, Transform, Load işlemleri

**Sonraki Bölümde:** Trigger yapıları ve transaction yönetimini öğreneceğiz.
