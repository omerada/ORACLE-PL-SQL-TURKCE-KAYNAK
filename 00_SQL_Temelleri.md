# SQL Temelleri - PL/SQL Öncesi Gereksinimler

## 1. SQL Nedir?

SQL (Structured Query Language), veritabanlarında veri sorgulamak, eklemek, güncellemek ve silmek için kullanılan standart dildir.

**Neden SQL Bilmek Gerekli?**

- PL/SQL'in temeli SQL'dir
- Her PL/SQL bloğunda SQL kullanılır
- Veritabanı işlemlerinin hepsinde SQL gerekir
- Performance optimizasyonu için SQL bilgisi şart

## 2. Temel SQL Komutları

### DDL (Data Definition Language) - Veri Tanımlama Dili

**Ne İşe Yarar:** Veritabanı nesnelerini (tablo, index, constraint vb.) oluşturmak, değiştirmek ve silmek için kullanılır.

Veritabanı yapısını oluşturan komutlar:

```sql
-- Tablo oluşturma: Çalışanlar tablosu örneği
-- PRIMARY KEY: Tablonun birincil anahtarı, her satırı benzersiz tanımlar
-- NOT NULL: Bu sütun boş bırakılamaz
-- UNIQUE: Bu sütunda tekrar eden değer olamaz
-- DEFAULT: Varsayılan değer atar
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
-- ADD: Tabloya yeni sütun ekler
ALTER TABLE employees ADD phone VARCHAR2(20);
-- MODIFY: Mevcut sütunun veri tipini veya boyutunu değiştirir
ALTER TABLE employees MODIFY salary NUMBER(10,2);
-- DROP COLUMN: Sütunu tablodan siler
ALTER TABLE employees DROP COLUMN phone;

-- Tablo silme
DROP TABLE employees;

-- Index oluşturma
CREATE INDEX idx_emp_dept ON employees(department_id);
CREATE INDEX idx_emp_name ON employees(first_name, last_name);
```

### DML (Data Manipulation Language) - Veri İşleme Dili

**Ne İşe Yarar:** Tablolardaki verileri eklemek, güncellemek, silmek için kullanılır.

Veri işleme komutları:

```sql
-- Veri ekleme
-- INSERT INTO: Tabloya yeni kayıt ekler
-- VALUES: Eklenecek değerleri belirtir
INSERT INTO employees (employee_id, first_name, last_name, email, salary, department_id)
VALUES (1, 'Ali', 'Veli', 'ali.veli@company.com', 5000, 10);

-- Çoklu veri ekleme
-- SELECT ile başka tablodan veri kopyalama
INSERT INTO employees (employee_id, first_name, last_name, email, salary, department_id)
SELECT emp_id, fname, lname, email_addr, sal, dept_id FROM temp_employees;

-- Veri güncelleme
-- UPDATE: Mevcut kayıtları değiştirir
-- SET: Hangi sütunların nasıl değişeceğini belirtir
-- WHERE: Hangi kayıtların güncelleneğini filtreler (dikkat: WHERE olmadan tüm kayıtlar güncellenir!)
UPDATE employees
SET salary = salary * 1.1
WHERE department_id = 10;

-- Birden fazla sütunu aynı anda güncelleme
UPDATE employees
SET salary = 6000, department_id = 20
WHERE employee_id = 1;-- Veri silme
-- DELETE: Kayıtları tablodan siler
-- WHERE kullanmadan DELETE tüm tabloyu boşaltır, dikkatli olun!
DELETE FROM employees WHERE employee_id = 1;
-- Belirli tarihten önce işe başlayan çalışanları sil
DELETE FROM employees WHERE hire_date < DATE '2020-01-01';
```

### DQL (Data Query Language) - Veri Sorgulama Dili

**Ne İşe Yarar:** Tablolardan veri sorgulamak, filtrelemek, sıralamak için kullanılır.

Veri sorgulama komutları:

```sql
-- Basit sorgular
-- SELECT *: Tüm sütunları getirir (performans için önerilmez)
SELECT * FROM employees;
-- Sadece belirli sütunları getir (daha hızlı)
SELECT first_name, last_name, salary FROM employees;
-- DISTINCT: Tekrar eden değerleri tek sefer gösterir
SELECT DISTINCT department_id FROM employees;

-- WHERE koşulları
-- Belirli koşulu sağlayan kayıtları filtreler
SELECT * FROM employees WHERE salary > 5000;
-- IN: Birden fazla değer içinde arama yapar
SELECT * FROM employees WHERE department_id IN (10, 20, 30);
-- LIKE: Metin arama yapar (% = herhangi bir karakter dizisi)
SELECT * FROM employees WHERE first_name LIKE 'A%';
-- BETWEEN: Belirli aralıktaki değerleri bulur
SELECT * FROM employees WHERE hire_date BETWEEN DATE '2020-01-01' AND DATE '2023-12-31';

-- Sıralama
-- ORDER BY: Sonuçları belirli sütuna göre sıralar
-- DESC: Büyükten küçüğe, ASC: Küçükten büyüğe (varsayılan)
SELECT * FROM employees ORDER BY salary DESC;
-- Birden fazla sütuna göre sıralama
SELECT * FROM employees ORDER BY department_id, salary DESC;

-- Gruplama
-- GROUP BY: Verileri gruplar ve toplam fonksiyonlar kullanır
-- COUNT: Sayım yapar, AVG: Ortalama, MAX/MIN: En büyük/küçük
-- HAVING: GROUP BY sonrası filtreleme yapar (WHERE group by öncesi için)
SELECT department_id, COUNT(*), AVG(salary), MAX(salary), MIN(salary)
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 5;
```

## 3. JOIN İşlemleri

**JOIN Nedir:** İki veya daha fazla tabloyu bağlantılı sütunlar üzerinden birleştirmek için kullanılır.

### INNER JOIN

**Ne İşe Yarar:** Sadece her iki tabloda eşleşen kayıtları getirir. En çok kullanılan JOIN tipi.

```sql
-- Çalışan ve departman bilgisini birleştir
-- Sadece departmanı olan çalışanlar gelir
SELECT
    e.employee_id,              -- Çalışan ID'si
    e.first_name,               -- Çalışan adı
    e.last_name,                -- Çalışan soyadı
    e.salary,                   -- Maaş
    d.department_name,          -- Departman adı
    d.location_id               -- Departman lokasyonu
FROM employees e               -- Ana tablo (alias: e)
INNER JOIN departments d       -- Birleştirilecek tablo (alias: d)
    ON e.department_id = d.department_id  -- Bağlantı koşulu
WHERE e.salary > 5000          -- Ek filtre
ORDER BY e.salary DESC;
```

### LEFT JOIN (LEFT OUTER JOIN)

**Ne İşe Yarar:** Sol tablodaki tüm kayıtları getirir, sağ tabloda eşleşme yoksa NULL değerler gösterir.

```sql
-- Tüm çalışanları getir, departmanı olmasa bile
-- Departmanı olmayan çalışanlar için department_name NULL olur
SELECT
    e.first_name,
    e.last_name,
    e.salary,
    d.department_name,
    CASE
        WHEN d.department_name IS NULL THEN 'Departman Atanmamış'
        ELSE d.department_name
    END as dept_status
FROM employees e
LEFT JOIN departments d
    ON e.department_id = d.department_id
ORDER BY d.department_name NULLS LAST;  -- NULL'lar en sona
```

### RIGHT JOIN (RIGHT OUTER JOIN)

**Ne İşe Yarar:** Sağ tablodaki tüm kayıtları getirir, sol tabloda eşleşme yoksa NULL değerler gösterir.

```sql
-- Tüm departmanları getir, çalışanı olmasa bile
-- Boş departmanlar için employee bilgileri NULL olur
SELECT
    e.first_name,
    e.last_name,
    e.salary,
    d.department_name,
    d.department_id
FROM employees e
RIGHT JOIN departments d
    ON e.department_id = d.department_id
ORDER BY d.department_name;
```

### FULL OUTER JOIN

**Ne İşe Yarar:** Her iki tablodaki tüm kayıtları getirir, eşleşme olmayan yerler NULL olur.

```sql
-- Hem çalışanı olmayan departmanları, hem departmanı olmayan çalışanları göster
SELECT
    e.employee_id,
    e.first_name,
    e.last_name,
    d.department_name,
    d.department_id,
    CASE
        WHEN e.employee_id IS NULL THEN 'Boş Departman'
        WHEN d.department_id IS NULL THEN 'Departmansız Çalışan'
        ELSE 'Normal Kayıt'
    END as record_type
FROM employees e
FULL OUTER JOIN departments d
    ON e.department_id = d.department_id
ORDER BY record_type, d.department_name;
```

### SELF JOIN

**Ne İşe Yarar:** Aynı tabloyu kendisiyle birleştirir. Genelde hiyerarşik veriler için kullanılır.

```sql
-- Çalışan ve yöneticisi ilişkisi
-- Her çalışanın manager_id'si başka bir çalışanın employee_id'sine eşit
SELECT
    e.employee_id as emp_id,
    e.first_name || ' ' || e.last_name as employee_name,
    e.salary as emp_salary,
    m.employee_id as mgr_id,
    m.first_name || ' ' || m.last_name as manager_name,
    m.salary as mgr_salary,
    (m.salary - e.salary) as salary_diff
FROM employees e                    -- Çalışan tablosu
LEFT JOIN employees m               -- Yönetici tablosu (aynı tablo)
    ON e.manager_id = m.employee_id -- Bağlantı: Çalışanın yöneticisi
WHERE e.employee_id != m.employee_id OR m.employee_id IS NULL  -- Kendini yönetmeyenler
ORDER BY m.last_name, e.last_name;

-- İki seviye hiyerarşi
SELECT
    e.first_name as employee,
    m.first_name as manager,
    gm.first_name as grand_manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id
LEFT JOIN employees gm ON m.manager_id = gm.employee_id;
```

### Multiple Table JOIN

```sql
-- Üç veya daha fazla tablo birleştirme
SELECT
    e.first_name,
    e.last_name,
    e.salary,
    d.department_name,
    l.city,
    l.country_id,
    c.country_name,
    j.job_title
FROM employees e
JOIN departments d ON e.department_id = d.department_id
JOIN locations l ON d.location_id = l.location_id
JOIN countries c ON l.country_id = c.country_id
JOIN jobs j ON e.job_id = j.job_id
WHERE e.salary > 10000
ORDER BY c.country_name, l.city, d.department_name;
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
    -- UPPER: Metni büyük harfe çevirir (Ali -> ALİ)
    UPPER(first_name) as upper_name,
    -- LOWER: Metni küçük harfe çevirir (VELİ -> veli)
    LOWER(last_name) as lower_name,
    -- INITCAP: Her kelimenin ilk harfini büyük yapar (ali veli -> Ali Veli)
    INITCAP(first_name || ' ' || last_name) as full_name,
    -- LENGTH: Metin uzunluğunu döner (karakter sayısı)
    LENGTH(first_name) as name_length,
    -- SUBSTR: Metinden belirli kısmı alır (başlangıç pozisyonu, uzunluk)
    -- INSTR: Bir karakterin metindeki pozisyonunu bulur
    SUBSTR(email, 1, INSTR(email, '@') - 1) as username,
    -- REPLACE: Metindeki karakteri başka karakterle değiştirir
    REPLACE(phone, '-', '.') as formatted_phone,
    -- TRIM: Başındaki ve sonundaki boşlukları temizler
    TRIM(first_name) as trimmed_name,
    -- LPAD: Soldan belirli karakterle doldurur (toplam uzunluk, dolgu karakteri)
    LPAD(employee_id, 6, '0') as padded_id,
    -- RPAD: Sağdan belirli karakterle doldurur
    RPAD(first_name, 20, '.') as right_padded,
    -- LTRIM: Soldaki belirtilen karakterleri siler
    LTRIM(phone, '0') as phone_without_leading_zero,
    -- RTRIM: Sağdaki belirtilen karakterleri siler
    RTRIM(email, '.com') as email_without_domain
FROM employees;
```

**Yaygın String Operasyonları:**

```sql
-- Concatenation (Birleştirme)
SELECT
    first_name || ' ' || last_name as full_name,
    CONCAT(CONCAT(first_name, ' '), last_name) as full_name_concat
FROM employees;

-- Pattern matching
SELECT * FROM employees
WHERE REGEXP_LIKE(email, '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

-- String replacement with regex
SELECT REGEXP_REPLACE(phone, '[^0-9]', '') as clean_phone
FROM employees;
```

### Numeric Fonksiyonları

```sql
SELECT
    salary,
    -- ROUND: Sayıyı yuvarlar (5432.67 -> 5400 when -2)
    ROUND(salary, -2) as rounded_salary,
    -- TRUNC: Sayıyı keser, yuvarlamaz (5432.67 -> 5000 when -3)
    TRUNC(salary, -3) as truncated_salary,
    -- CEIL: Yukarı yuvarlar (5.1 -> 6)
    CEIL(salary/1000) as ceiling_thousands,
    -- FLOOR: Aşağı yuvarlar (5.9 -> 5)
    FLOOR(salary/1000) as floor_thousands,
    -- MOD: Bölme işleminden kalanı döner (10 MOD 3 = 1)
    MOD(employee_id, 2) as even_odd,  -- 0=çift, 1=tek
    -- ABS: Mutlak değer döner (negatif sayıları pozitif yapar)
    ABS(salary - 5000) as salary_diff,
    -- POWER: Üs alma (2^3 = 8)
    POWER(2, 3) as power_example,
    -- SQRT: Karekök alma
    SQRT(25) as square_root,
    -- SIGN: Sayının işaretini döner (-1, 0, 1)
    SIGN(salary - 5000) as salary_sign
FROM employees;

-- Mathematical constants and functions
SELECT
    -- PI sayısı
    ACOS(-1) as pi_value,
    -- Trigonometric functions
    SIN(ACOS(-1)/2) as sin_90_degrees,  -- sin(π/2) = 1
    COS(0) as cos_0_degrees,            -- cos(0) = 1
    -- Logarithmic functions
    LN(2.71828) as natural_log,         -- doğal logaritma
    LOG(10, 100) as log_base_10         -- 10 tabanında logaritma
FROM dual;
```

### Date Fonksiyonları

```sql
SELECT
    -- SYSDATE: Sistemin şu anki tarih ve saatini döner
    SYSDATE as current_date,
    -- ADD_MONTHS: Tarihe ay ekler (negatif değerle çıkarır)
    ADD_MONTHS(hire_date, 12) as anniversary,
    -- MONTHS_BETWEEN: İki tarih arasındaki ay farkını hesaplar
    MONTHS_BETWEEN(SYSDATE, hire_date) as months_worked,
    -- EXTRACT: Tarihten belirli bileşeni çıkarır (YEAR, MONTH, DAY)
    EXTRACT(YEAR FROM hire_date) as hire_year,
    EXTRACT(MONTH FROM hire_date) as hire_month,
    EXTRACT(DAY FROM hire_date) as hire_day,
    -- TO_CHAR: Tarihi belirli formatta string'e çevirir
    TO_CHAR(hire_date, 'DD/MM/YYYY') as formatted_date,
    TO_CHAR(hire_date, 'DAY, DD MONTH YYYY') as long_format,
    -- TRUNC: Tarihi belirli seviyeye yuvarlar
    TRUNC(hire_date, 'MONTH') as month_start,  -- Ayın ilk günü
    TRUNC(hire_date, 'YEAR') as year_start,    -- Yılın ilk günü
    -- LAST_DAY: Ayın son gününü döner
    LAST_DAY(hire_date) as month_end,
    -- NEXT_DAY: Belirtilen günün bir sonraki oluşumunu bulur
    NEXT_DAY(hire_date, 'MONDAY') as next_monday,
    -- Tarih aritmetiği
    hire_date + 30 as thirty_days_later,
    hire_date - 7 as week_before
FROM employees;

-- Tarih formatları
SELECT
    TO_CHAR(SYSDATE, 'DD/MM/YYYY HH24:MI:SS') as date_time,
    TO_CHAR(SYSDATE, 'Day, DD Month YYYY') as readable_date,
    TO_CHAR(SYSDATE, 'Q') as quarter,          -- Çeyrek (1-4)
    TO_CHAR(SYSDATE, 'WW') as week_of_year,    -- Yılın kaçıncı haftası
    TO_CHAR(SYSDATE, 'DDD') as day_of_year     -- Yılın kaçıncı günü
FROM dual;

-- String'den tarihe çevirme
SELECT
    TO_DATE('15/01/2024', 'DD/MM/YYYY') as parsed_date,
    TO_DATE('2024-01-15 14:30:00', 'YYYY-MM-DD HH24:MI:SS') as parsed_datetime
FROM dual;
```

### Conditional Fonksiyonları

```sql
SELECT
    employee_id,
    first_name,
    salary,

    -- CASE: Çoklu koşul kontrolü (if-else if-else yapısı gibi)
    CASE
        WHEN salary > 15000 THEN 'Senior Level'     -- Eğer maaş > 15000
        WHEN salary > 10000 THEN 'Mid Level'        -- Değilse eğer > 10000
        WHEN salary > 5000 THEN 'Junior Level'      -- Değilse eğer > 5000
        ELSE 'Entry Level'                          -- Hiçbiri değilse
    END as salary_grade,

    -- CASE (Simple form): Belirli değerlere göre karşılaştırma
    CASE department_id
        WHEN 10 THEN 'Finans'                      -- department_id = 10 ise
        WHEN 20 THEN 'Pazarlama'                    -- department_id = 20 ise
        WHEN 30 THEN 'IT'                          -- department_id = 30 ise
        ELSE 'Diğer Departman'                     -- Hiçbiri değilse
    END as dept_name_tr,

    -- DECODE: Oracle'a özgü, CASE'in kısa hali
    -- DECODE(kontrol_edilen_değer, değer1, sonuç1, değer2, sonuç2, ..., varsayılan)
    DECODE(department_id,
           10, 'Finance',               -- 10 ise Finance
           20, 'Marketing',             -- 20 ise Marketing
           30, 'IT',                    -- 30 ise IT
           'Other') as dept_name,       -- Hiçbiri değilse Other

    -- Gender decode örneği
    DECODE(SUBSTR(first_name, -1, 1),
           'a', 'Female',               -- İsim 'a' ile bitiyorsa kadın
           'e', 'Female',               -- İsim 'e' ile bitiyorsa kadın
           'Male') as estimated_gender, -- Diğer durumlarda erkek

    -- NVL: NULL değerleri değiştir (null ise ikinci değeri kullan)
    NVL(commission_pct, 0) as commission,  -- NULL ise 0 yap
    NVL(phone_number, 'Telefon Yok') as phone_display,

    -- NVL2: NULL olup olmamasına göre farklı değerler döner
    -- NVL2(kontrol_değeri, NULL_değilse_bu, NULL_ise_bu)
    NVL2(commission_pct, 'Komisyon Var', 'Komisyon Yok') as comm_status,
    NVL2(manager_id, 'Has Manager', 'Top Level') as manager_status,

    -- COALESCE: Birden fazla değerden ilk NULL olmayanı döner
    COALESCE(commission_pct, bonus_pct, 0) as incentive,  -- İlk NULL olmayanı al
    COALESCE(email, phone_number, 'No Contact') as primary_contact,

    -- NULLIF: İki değer eşitse NULL, değilse ilki döner
    NULLIF(salary, 0) as salary_check,    -- Maaş 0 ise NULL yap
    NULLIF(department_id, 999) as valid_dept_id,  -- 999 ise NULL yap

    -- GREATEST ve LEAST: En büyük ve en küçük değer
    GREATEST(salary, commission_pct * 100, 1000) as max_value,
    LEAST(salary, 50000) as capped_salary,  -- Maaş 50000'i geçmesin

    -- Karmaşık CASE örneği
    CASE
        WHEN salary IS NULL THEN 'Maaş Belirsiz'
        WHEN salary = 0 THEN 'Maaşsız'
        WHEN MOD(salary, 1000) = 0 THEN 'Yuvarlanmış Maaş'
        ELSE 'Normal Maaş'
    END as salary_type
FROM employees
WHERE employee_id <= 110;  -- Sonuçları sınırla

-- Conditional aggregate örneği
SELECT
    department_id,
    COUNT(*) as total_employees,
    -- Koşullu sayma
    COUNT(CASE WHEN salary > 10000 THEN 1 END) as high_earners,
    COUNT(CASE WHEN commission_pct IS NOT NULL THEN 1 END) as commissioned_emp,
    -- Koşullu toplam
    SUM(CASE WHEN salary > 10000 THEN salary ELSE 0 END) as high_earner_total,
    -- Koşullu ortalama
    AVG(CASE WHEN hire_date > DATE '2005-01-01' THEN salary END) as new_emp_avg_sal
FROM employees
GROUP BY department_id
ORDER BY department_id;
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

### 1. Performance

- **SELECT \* yerine gerekli sütunları belirtin**

  ```sql
  -- KÖTÜ: Tüm sütunları getir (yavaş, network trafiği fazla)
  SELECT * FROM employees WHERE department_id = 10;

  -- İYİ: Sadece gerekli sütunları getir
  SELECT employee_id, first_name, last_name, salary
  FROM employees WHERE department_id = 10;
  ```

- **WHERE koşullarında index'li sütunları kullanın**

  ```sql
  -- İYİ: employee_id PRIMARY KEY olduğu için index var
  SELECT * FROM employees WHERE employee_id = 100;

  -- KÖTÜ: first_name genelde index'siz
  SELECT * FROM employees WHERE UPPER(first_name) = 'JOHN';
  ```

- **LIMIT/ROWNUM kullanarak büyük sonuçları sınırlayın**

  ```sql
  -- Oracle: ROWNUM ile sınırlama
  SELECT * FROM employees WHERE ROWNUM <= 10;

  -- Oracle 12c+: FETCH FIRST ile sınırlama (modern yöntem)
  SELECT * FROM employees ORDER BY salary DESC
  FETCH FIRST 10 ROWS ONLY;
  ```

### 2. Readability (Okunabilirlik)

- **Anlamlı alias'lar kullanın**

  ```sql
  SELECT
      e.employee_id as emp_id,
      e.first_name as name,
      d.department_name as dept
  FROM employees e
  JOIN departments d ON e.department_id = d.department_id;
  ```

- **SQL'i düzgün formatlayın**

  ```sql
  -- İYİ: Okunabilir format
  SELECT e.first_name,
         e.last_name,
         e.salary,
         d.department_name
  FROM employees e
  JOIN departments d
    ON e.department_id = d.department_id
  WHERE e.salary > 5000
  ORDER BY e.salary DESC;
  ```

- **Karmaşık sorguları CTE ile basitleştirin**
  ```sql
  WITH high_earners AS (
      SELECT * FROM employees WHERE salary > 10000
  ),
  dept_summary AS (
      SELECT department_id, AVG(salary) as avg_sal
      FROM high_earners
      GROUP BY department_id
  )
  SELECT he.first_name, he.salary, ds.avg_sal
  FROM high_earners he
  JOIN dept_summary ds ON he.department_id = ds.department_id;
  ```

### 3. Security

- **Parameterized queries kullanın (SQL Injection korunması)**

  ```sql
  -- Java/C# gibi uygulamalarda:
  -- PreparedStatement ps = connection.prepareStatement(
  --     "SELECT * FROM employees WHERE employee_id = ?");
  -- ps.setInt(1, employeeId);
  ```

- **Least privilege principle'ı uygulayın**

  ```sql
  -- Kullanıcıya sadece gerekli yetkileri verin
  GRANT SELECT ON employees TO readonly_user;
  GRANT SELECT, INSERT, UPDATE ON employees TO data_entry_user;
  ```

- **Sensitive data'yı maskeleyetin**
  ```sql
  -- Credit card numaralarını maskele
  SELECT employee_id,
         first_name,
         '****-****-****-' || SUBSTR(credit_card, -4) as masked_card
  FROM employee_sensitive_data;
  ```

### 4. Data Integrity (Veri Bütünlüğü)

- **Constraints kullanın**

  ```sql
  CREATE TABLE employees (
      employee_id NUMBER PRIMARY KEY,
      email VARCHAR2(100) UNIQUE NOT NULL,
      salary NUMBER CHECK (salary > 0),
      hire_date DATE DEFAULT SYSDATE
  );
  ```

- **Foreign Keys ile referential integrity sağlayın**
  ```sql
  ALTER TABLE employees
  ADD CONSTRAINT fk_emp_dept
  FOREIGN KEY (department_id) REFERENCES departments(department_id);
  ```

### 5. Common SQL Anti-Patterns (Kaçınılması Gerekenler)

```sql
-- KÖTÜ: IN clause'da çok fazla değer
SELECT * FROM employees
WHERE employee_id IN (1,2,3,4,5,...,1000);  -- Yavaş

-- İYİ: Temporary table veya EXISTS kullan
WITH temp_ids AS (
    SELECT 1 as id FROM dual UNION ALL
    SELECT 2 FROM dual UNION ALL
    SELECT 3 FROM dual
)
SELECT e.* FROM employees e
JOIN temp_ids t ON e.employee_id = t.id;

-- KÖTÜ: SELECT DISTINCT sık kullanımı
SELECT DISTINCT e.first_name, d.department_name
FROM employees e, departments d;  -- Cartesian product

-- İYİ: Proper JOIN kullan
SELECT e.first_name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

-- KÖTÜ: Fonksiyon WHERE clause'da
SELECT * FROM employees
WHERE UPPER(first_name) = 'JOHN';  -- Index kullanamaz

-- İYİ: Veriyi önceden normalize et veya functional index oluştur
CREATE INDEX idx_emp_upper_name ON employees(UPPER(first_name));
SELECT * FROM employees WHERE UPPER(first_name) = 'JOHN';
```

## Helpful SQL Utilities

### 1. Data Dictionary Views

```sql
-- Tablolar hakkında bilgi
SELECT table_name, num_rows, last_analyzed
FROM user_tables
WHERE table_name LIKE 'EMP%';

-- Sütun bilgileri
SELECT column_name, data_type, nullable, default_value
FROM user_tab_columns
WHERE table_name = 'EMPLOYEES'
ORDER BY column_id;

-- Index bilgileri
SELECT index_name, uniqueness, status
FROM user_indexes
WHERE table_name = 'EMPLOYEES';

-- Constraint bilgileri
SELECT constraint_name, constraint_type, status
FROM user_constraints
WHERE table_name = 'EMPLOYEES';
```

### 2. Explain Plan

```sql
-- Query execution plan'ı görmek için
EXPLAIN PLAN FOR
SELECT e.first_name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary > 5000;

-- Plan'ı görüntüle
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### 3. SQL Trace

```sql
-- Session'ı trace et (performance analizi için)
ALTER SESSION SET SQL_TRACE = TRUE;
-- Sorguları çalıştır
ALTER SESSION SET SQL_TRACE = FALSE;
```

**Sonraki Bölümde:** Oracle Database temel kavramlarını öğreneceğiz.
