# SQL Temelleri - PL/SQL Öncesi Gereksinimler

## 1. SQL Nedir?

SQL (Structured Query Language), veritabanlarında veri sorgulamak, eklemek, güncellemek ve silmek için kullanılan standart dildir.

**Neden SQL Bilmek Gerekli?**

- PL/SQL'in temeli SQL'dir
- Her PL/SQL bloğunda SQL kullanılır
- Veritabanı işlemlerinin hepsinde SQL gerekir
- Performance optimizasyonu için SQL bilgisi şart

## 2. Temel SQL Komutları

### DDL (Data Definition Language)

Veritabanı yapısını oluşturan komutlar:

```sql
-- Tablo oluşturma
CREATE TABLE employees (
    employee_id NUMBER(10) PRIMARY KEY,
    first_name VARCHAR2(50) NOT NULL,
    last_name VARCHAR2(50) NOT NULL,
    email VARCHAR2(100) UNIQUE,
    hire_date DATE DEFAULT SYSDATE,
    salary NUMBER(8,2),
    department_id NUMBER(10),
    manager_id NUMBER(10)
);

-- Tablo yapısını değiştirme
ALTER TABLE employees ADD phone VARCHAR2(20);
ALTER TABLE employees MODIFY salary NUMBER(10,2);
ALTER TABLE employees DROP COLUMN phone;

-- Tablo silme
DROP TABLE employees;

-- Index oluşturma
CREATE INDEX idx_emp_dept ON employees(department_id);
CREATE INDEX idx_emp_name ON employees(first_name, last_name);
```

### DML (Data Manipulation Language)

Veri işleme komutları:

```sql
-- Veri ekleme
INSERT INTO employees (employee_id, first_name, last_name, email, salary, department_id)
VALUES (1, 'Ali', 'Veli', 'ali.veli@company.com', 5000, 10);

-- Çoklu veri ekleme
INSERT INTO employees (employee_id, first_name, last_name, email, salary, department_id)
SELECT emp_id, fname, lname, email_addr, sal, dept_id FROM temp_employees;

-- Veri güncelleme
UPDATE employees
SET salary = salary * 1.1
WHERE department_id = 10;

UPDATE employees
SET salary = 6000, department_id = 20
WHERE employee_id = 1;

-- Veri silme
DELETE FROM employees WHERE employee_id = 1;
DELETE FROM employees WHERE hire_date < DATE '2020-01-01';
```

### DQL (Data Query Language)

Veri sorgulama komutları:

```sql
-- Basit sorgular
SELECT * FROM employees;
SELECT first_name, last_name, salary FROM employees;
SELECT DISTINCT department_id FROM employees;

-- WHERE koşulları
SELECT * FROM employees WHERE salary > 5000;
SELECT * FROM employees WHERE department_id IN (10, 20, 30);
SELECT * FROM employees WHERE first_name LIKE 'A%';
SELECT * FROM employees WHERE hire_date BETWEEN DATE '2020-01-01' AND DATE '2023-12-31';

-- Sıralama
SELECT * FROM employees ORDER BY salary DESC;
SELECT * FROM employees ORDER BY department_id, salary DESC;

-- Gruplama
SELECT department_id, COUNT(*), AVG(salary), MAX(salary), MIN(salary)
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 5;
```

## 3. JOIN İşlemleri

### INNER JOIN

```sql
SELECT e.first_name, e.last_name, d.department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id;
```

### LEFT JOIN

```sql
SELECT e.first_name, e.last_name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;
```

### RIGHT JOIN

```sql
SELECT e.first_name, e.last_name, d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.department_id;
```

### FULL OUTER JOIN

```sql
SELECT e.first_name, e.last_name, d.department_name
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.department_id;
```

### SELF JOIN

```sql
-- Çalışan ve yöneticisi
SELECT e.first_name || ' ' || e.last_name as employee,
       m.first_name || ' ' || m.last_name as manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id;
```

## 4. Alt Sorgular (Subqueries)

### Scalar Subquery

```sql
SELECT first_name, last_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

### Multiple Row Subquery

```sql
SELECT first_name, last_name, department_id
FROM employees
WHERE department_id IN (
    SELECT department_id
    FROM departments
    WHERE location_id = 1700
);
```

### Correlated Subquery

```sql
SELECT e1.first_name, e1.last_name, e1.salary
FROM employees e1
WHERE e1.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e1.department_id
);
```

### EXISTS

```sql
SELECT d.department_name
FROM departments d
WHERE EXISTS (
    SELECT 1 FROM employees e
    WHERE e.department_id = d.department_id
);
```

## 5. Oracle Fonksiyonları

### String Fonksiyonları

```sql
SELECT
    UPPER(first_name) as upper_name,
    LOWER(last_name) as lower_name,
    INITCAP(first_name || ' ' || last_name) as full_name,
    LENGTH(first_name) as name_length,
    SUBSTR(email, 1, INSTR(email, '@') - 1) as username,
    REPLACE(phone, '-', '.') as formatted_phone,
    TRIM(first_name) as trimmed_name
FROM employees;
```

### Numeric Fonksiyonları

```sql
SELECT
    ROUND(salary, -2) as rounded_salary,
    TRUNC(salary, -3) as truncated_salary,
    CEIL(salary/1000) as ceiling_thousands,
    FLOOR(salary/1000) as floor_thousands,
    MOD(employee_id, 2) as even_odd,
    ABS(salary - 5000) as salary_diff
FROM employees;
```

### Date Fonksiyonları

```sql
SELECT
    SYSDATE as current_date,
    ADD_MONTHS(hire_date, 12) as anniversary,
    MONTHS_BETWEEN(SYSDATE, hire_date) as months_worked,
    EXTRACT(YEAR FROM hire_date) as hire_year,
    TO_CHAR(hire_date, 'DD/MM/YYYY') as formatted_date,
    TRUNC(hire_date, 'MONTH') as month_start,
    LAST_DAY(hire_date) as month_end
FROM employees;
```

### Conditional Fonksiyonları

```sql
SELECT
    first_name,
    salary,
    CASE
        WHEN salary > 10000 THEN 'High'
        WHEN salary > 5000 THEN 'Medium'
        ELSE 'Low'
    END as salary_grade,

    DECODE(department_id,
           10, 'Finance',
           20, 'Marketing',
           30, 'IT',
           'Other') as dept_name,

    NVL(commission_pct, 0) as commission,
    NVL2(commission_pct, 'Has Commission', 'No Commission') as comm_status,
    COALESCE(commission_pct, bonus_pct, 0) as incentive
FROM employees;
```

## 6. Aggregate Fonksiyonları

```sql
-- Temel aggregate
SELECT
    COUNT(*) as total_employees,
    COUNT(commission_pct) as employees_with_commission,
    SUM(salary) as total_salary,
    AVG(salary) as average_salary,
    MAX(salary) as highest_salary,
    MIN(salary) as lowest_salary,
    STDDEV(salary) as salary_stddev
FROM employees;

-- Analytic functions
SELECT
    first_name,
    last_name,
    salary,
    RANK() OVER (ORDER BY salary DESC) as salary_rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) as salary_dense_rank,
    ROW_NUMBER() OVER (ORDER BY salary DESC) as row_num,
    NTILE(4) OVER (ORDER BY salary) as salary_quartile,
    LAG(salary) OVER (ORDER BY hire_date) as prev_emp_salary,
    LEAD(salary) OVER (ORDER BY hire_date) as next_emp_salary
FROM employees;
```

## 7. Window Functions

```sql
-- Departman bazında analiz
SELECT
    first_name,
    last_name,
    department_id,
    salary,
    AVG(salary) OVER (PARTITION BY department_id) as dept_avg_salary,
    MAX(salary) OVER (PARTITION BY department_id) as dept_max_salary,
    COUNT(*) OVER (PARTITION BY department_id) as dept_emp_count,
    RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) as dept_salary_rank
FROM employees;

-- Running totals
SELECT
    hire_date,
    first_name,
    salary,
    SUM(salary) OVER (ORDER BY hire_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as running_total,
    AVG(salary) OVER (ORDER BY hire_date ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING) as moving_avg
FROM employees
ORDER BY hire_date;
```

## 8. CTE (Common Table Expressions)

```sql
-- Recursive CTE - Organizational hierarchy
WITH emp_hierarchy (employee_id, first_name, last_name, manager_id, level_num, path) AS (
    -- Anchor: Top level managers
    SELECT employee_id, first_name, last_name, manager_id, 1, first_name
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: Direct reports
    SELECT e.employee_id, e.first_name, e.last_name, e.manager_id,
           eh.level_num + 1, eh.path || ' -> ' || e.first_name
    FROM employees e
    JOIN emp_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM emp_hierarchy ORDER BY level_num, path;

-- Non-recursive CTE
WITH high_earners AS (
    SELECT * FROM employees WHERE salary > 8000
),
dept_stats AS (
    SELECT department_id, AVG(salary) as avg_sal
    FROM high_earners
    GROUP BY department_id
)
SELECT he.first_name, he.salary, ds.avg_sal
FROM high_earners he
JOIN dept_stats ds ON he.department_id = ds.department_id;
```

## 9. Set Operations

```sql
-- UNION - Combine results, remove duplicates
SELECT first_name, last_name FROM employees WHERE department_id = 10
UNION
SELECT first_name, last_name FROM employees WHERE salary > 10000;

-- UNION ALL - Combine results, keep duplicates
SELECT department_id FROM employees WHERE salary > 5000
UNION ALL
SELECT department_id FROM employees WHERE hire_date > DATE '2020-01-01';

-- INTERSECT - Common records
SELECT employee_id FROM employees WHERE department_id = 10
INTERSECT
SELECT employee_id FROM employees WHERE salary > 5000;

-- MINUS - Records in first query but not in second
SELECT employee_id FROM employees
MINUS
SELECT employee_id FROM employees WHERE salary > 10000;
```

## 10. Transaction Control

```sql
-- Transaction başlatma ve yönetimi
BEGIN
    INSERT INTO employees (employee_id, first_name, last_name, email)
    VALUES (1001, 'Test', 'User', 'test@company.com');

    UPDATE employees SET salary = salary * 1.1 WHERE department_id = 10;

    SAVEPOINT before_delete;

    DELETE FROM employees WHERE employee_id = 999;

    -- Hata durumunda sadece delete'i geri al
    ROLLBACK TO before_delete;

    -- Tüm değişiklikleri kaydet
    COMMIT;
END;
/
```

## 11. Views

```sql
-- Basit view
CREATE VIEW emp_dept_view AS
SELECT e.employee_id, e.first_name, e.last_name, e.salary,
       d.department_name, d.location_id
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

-- Complex view with calculations
CREATE VIEW salary_analysis AS
SELECT
    department_id,
    COUNT(*) as emp_count,
    AVG(salary) as avg_salary,
    MAX(salary) as max_salary,
    MIN(salary) as min_salary,
    STDDEV(salary) as salary_stddev
FROM employees
GROUP BY department_id;

-- Materialized view (performance için)
CREATE MATERIALIZED VIEW mv_dept_summary
REFRESH FAST ON COMMIT
AS
SELECT department_id, COUNT(*) as emp_count, SUM(salary) as total_salary
FROM employees
GROUP BY department_id;
```

## 12. Sequences

```sql
-- Sequence oluşturma
CREATE SEQUENCE emp_seq
START WITH 1000
INCREMENT BY 1
MAXVALUE 999999
NOCYCLE
CACHE 20;

-- Sequence kullanımı
INSERT INTO employees (employee_id, first_name, last_name)
VALUES (emp_seq.NEXTVAL, 'John', 'Doe');

-- Sequence bilgileri
SELECT emp_seq.CURRVAL FROM DUAL; -- Son kullanılan değer
SELECT emp_seq.NEXTVAL FROM DUAL; -- Sonraki değer
```

## Pratik Egzersizler

### Temel Seviye

1. Tüm çalışanları maaşlarına göre sıralayın
2. Departman başına çalışan sayısını bulun
3. En yüksek maaşlı 5 çalışanı listeleyin
4. Belirli bir tarihten sonra işe başlayan çalışanları bulun

### Orta Seviye

1. Her departmandaki ortalama maaşın üstünde maaş alan çalışanları bulun
2. Hiç çalışanı olmayan departmanları listeleyin
3. Her çalışanın maaş sıralamasını departman içinde hesaplayın
4. Son 3 ayda işe başlayan çalışanları listeleyin

### İleri Seviye

1. Organizasyonel hiyerarşiyi göster (manager-employee ilişkisi)
2. Her departmanın toplam maaş bütçesini hesaplayın
3. Çalışanların yıllık maaş artış trendlerini analiz edin
4. Departmanlar arası maaş dağılımını analiz edin

## SQL Best Practices

1. **Performance**

   - SELECT \* yerine gerekli sütunları belirtin
   - WHERE koşullarında index'li sütunları kullanın
   - LIMIT/ROWNUM kullanarak büyük sonuçları sınırlayın

2. **Readability**

   - Anlamlı alias'lar kullanın
   - SQL'i düzgün formatlayın
   - Karmaşık sorguları CTE ile basitleştirin

3. **Security**
   - Parameterized queries kullanın
   - Least privilege principle'ı uygulayın
   - Sensitive data'yı maskeleyeyin

**Sonraki Bölümde:** Oracle Database temel kavramlarını öğreneceğiz.
