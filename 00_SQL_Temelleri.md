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

-- TRUNCATE: Tablonun tüm içeriğini hızlıca siler
-- DELETE'den farklı olarak:
-- 1. WHERE koşulu kullanamaz (tüm tablo silinir)
-- 2. Rollback yapılamaz (DDL komutu)
-- 3. Çok daha hızlıdır (log tutmaz)
-- 4. AUTO_INCREMENT/SEQUENCE sayaçlarını sıfırlar
TRUNCATE TABLE employees;

-- TRUNCATE vs DELETE karşılaştırması:
-- DELETE: WHERE kullanabilir, yavaş, rollback edilebilir, trigger çalışır
-- TRUNCATE: WHERE yok, hızlı, rollback edilemez, trigger çalışmaz
```

### DCL (Data Control Language) - Veri Kontrol Dili

**Ne İşe Yarar:** Veritabanı erişim yetkilerini yönetmek için kullanılır.

```sql
-- GRANT: Yetki verme
-- Kullanıcıya tablo üzerinde okuma yetkisi ver
GRANT SELECT ON employees TO hr_user;

-- Birden fazla yetki verme
GRANT SELECT, INSERT, UPDATE ON employees TO data_entry_user;

-- Tüm yetkiler verme
GRANT ALL PRIVILEGES ON employees TO admin_user;

-- Role oluşturma ve yetki verme
CREATE ROLE employee_role;
GRANT SELECT, INSERT ON employees TO employee_role;
GRANT employee_role TO hr_user;

-- Belirli sütunlara yetki verme
GRANT UPDATE (salary, department_id) ON employees TO hr_manager;

-- REVOKE: Yetki geri alma
-- Kullanıcıdan okuma yetkisini geri al
REVOKE SELECT ON employees FROM hr_user;

-- Birden fazla yetkiyi geri alma
REVOKE INSERT, UPDATE ON employees FROM data_entry_user;

-- Role'den yetki geri alma
REVOKE employee_role FROM hr_user;

-- WITH GRANT OPTION: Başkasına yetki verme hakkı
GRANT SELECT ON employees TO manager_user WITH GRANT OPTION;
-- manager_user artık başkalarına da SELECT yetkisi verebilir

-- CASCADE ile yetki geri alma
REVOKE SELECT ON employees FROM manager_user CASCADE;
-- manager_user'in başkalarına verdiği yetkiler de geri alınır

-- System privileges (sistem yetkileri)
GRANT CREATE TABLE TO developer_user;
GRANT CREATE VIEW TO analyst_user;
GRANT CREATE PROCEDURE TO app_developer;

-- Public'e yetki verme
GRANT SELECT ON departments TO PUBLIC;
-- Tüm kullanıcılar departments tablosunu okuyabilir

-- Yetki sorgulama
-- Kullanıcının yetkilerini görüntüleme
SELECT * FROM user_tab_privs;      -- Tablo yetkileri
SELECT * FROM user_sys_privs;      -- Sistem yetkileri
SELECT * FROM user_role_privs;     -- Role yetkileri

-- Başkalarına verilen yetkileri görme
SELECT * FROM user_tab_privs_made; -- Bu kullanıcının verdiği yetkiler
```

### TCL (Transaction Control Language) - Transaction Kontrol Dili

**Ne İşe Yarar:** Veritabanı transaction'larını yönetmek için kullanılır.

```sql
-- COMMIT: Değişiklikleri kalıcı olarak kaydet
INSERT INTO employees (employee_id, first_name) VALUES (1001, 'John');
UPDATE employees SET salary = 5000 WHERE employee_id = 1001;
COMMIT; -- İki işlem de kaydedildi

-- ROLLBACK: Değişiklikleri geri al
INSERT INTO employees (employee_id, first_name) VALUES (1002, 'Jane');
DELETE FROM employees WHERE employee_id = 100;
ROLLBACK; -- İki işlem de geri alındı

-- SAVEPOINT: Ara kaydetme noktası oluştur
INSERT INTO employees (employee_id, first_name) VALUES (1003, 'Bob');
SAVEPOINT sp1;
UPDATE employees SET salary = 6000 WHERE employee_id = 1003;
SAVEPOINT sp2;
DELETE FROM employees WHERE employee_id = 1003;

-- ROLLBACK TO: Belirli savepoint'e geri dön
ROLLBACK TO sp2; -- Sadece DELETE geri alınır, INSERT ve UPDATE kalur
ROLLBACK TO sp1; -- UPDATE ve DELETE geri alınır, sadece INSERT kalur

-- Automatic COMMIT scenarios
-- DDL komutları otomatik COMMIT yapar
INSERT INTO employees (employee_id, first_name) VALUES (1004, 'Alice');
CREATE TABLE test_table (id NUMBER); -- Otomatik COMMIT (üsteki INSERT kaydedildi)

-- Normal session sonu da otomatik COMMIT yapar
-- EXIT veya disconnect otomatik COMMIT tetikler

-- SET TRANSACTION: Transaction özelliklerini ayarla
SET TRANSACTION ISOLATION LEVEL READ_COMMITTED;
SET TRANSACTION READ_ONLY;  -- Sadece okuma için

-- Oracle'da implicit transaction başlatma
-- İlk DML komutu otomatik olarak transaction başlatır
-- Explicit BEGIN TRANSACTION yoktur (SQL Server'dan farklı)
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

**Ne İşe Yarar:** Aggregate fonksiyonlar birden fazla satırı bir araya getirip tek bir sonuç üretir.

```sql
-- Temel aggregate fonksiyonlar
SELECT
    -- COUNT(*): Tüm satırları sayar (NULL'lar dahil)
    COUNT(*) as total_employees,
    -- COUNT(column): Belirli sütundaki NULL olmayan değerleri sayar
    COUNT(commission_pct) as employees_with_commission,
    -- SUM: Sayısal değerlerin toplamını hesaplar
    SUM(salary) as total_salary,
    -- AVG: Aritmetik ortalama hesaplar (NULL'lar hariç)
    AVG(salary) as average_salary,
    -- MAX: En büyük değeri bulur
    MAX(salary) as highest_salary,
    -- MIN: En küçük değeri bulur
    MIN(salary) as lowest_salary,
    -- STDDEV: Standart sapma hesaplar
    STDDEV(salary) as salary_stddev,
    -- VARIANCE: Varyans hesaplar
    VARIANCE(salary) as salary_variance,
    -- Tarih fonksiyonları ile
    MIN(hire_date) as first_hire_date,
    MAX(hire_date) as last_hire_date
FROM employees;

-- Gruplanmış aggregate örnekleri
SELECT
    department_id,
    COUNT(*) as emp_count,
    SUM(salary) as dept_total_salary,
    AVG(salary) as dept_avg_salary,
    MIN(salary) as min_salary,
    MAX(salary) as max_salary,
    -- Koşullu aggregate
    COUNT(CASE WHEN salary > 10000 THEN 1 END) as high_earners,
    SUM(CASE WHEN commission_pct IS NOT NULL THEN salary ELSE 0 END) as commissioned_salary
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 1  -- En az 2 çalışanı olan departmanlar
ORDER BY dept_avg_salary DESC;
```

### Analytic Functions (Pencere Fonksiyonları)

**Ne İşe Yarar:** Her satır için hesaplama yapar ama GROUP BY gibi satırları birleştirmez.

```sql
-- Ranking fonksiyonları
SELECT
    employee_id,
    first_name,
    last_name,
    salary,
    department_id,

    -- RANK: Eşit değerler için aynı sıra, sonraki sıra atlanır (1,1,3,4)
    RANK() OVER (ORDER BY salary DESC) as salary_rank,

    -- DENSE_RANK: Eşit değerler için aynı sıra, sonraki sıra atlanmaz (1,1,2,3)
    DENSE_RANK() OVER (ORDER BY salary DESC) as salary_dense_rank,

    -- ROW_NUMBER: Her satıra benzersiz numara verir (1,2,3,4)
    ROW_NUMBER() OVER (ORDER BY salary DESC) as row_num,

    -- NTILE: Verileri belirtilen sayıda eşit gruba böler
    NTILE(4) OVER (ORDER BY salary) as salary_quartile,  -- 4 çeyrekliğe böl

    -- LAG: Önceki satırdaki değeri getirir
    LAG(salary, 1, 0) OVER (ORDER BY hire_date) as prev_emp_salary,

    -- LEAD: Sonraki satırdaki değeri getirir
    LEAD(salary, 1, 0) OVER (ORDER BY hire_date) as next_emp_salary,

    -- FIRST_VALUE: Partition'daki ilk değeri getirir
    FIRST_VALUE(salary) OVER (ORDER BY hire_date) as first_hired_salary,

    -- LAST_VALUE: Partition'daki son değeri getirir
    LAST_VALUE(salary) OVER (ORDER BY hire_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) as last_hired_salary
FROM employees
ORDER BY salary DESC;
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

**Ne İşe Yarar:** CTE, karmaşık sorguları basitleştirmek için geçici "view" benzeri yapılar oluşturur.

```sql
-- Recursive CTE - Organizational hierarchy
-- RECURSIVE: Kendini referans eden CTE (hiyerarşik veriler için)
WITH emp_hierarchy (employee_id, first_name, last_name, manager_id, level_num, path) AS (
    -- Anchor member: Başlangıç noktası (en üst seviye yöneticiler)
    SELECT
        employee_id,
        first_name,
        last_name,
        manager_id,
        1 as level_num,                    -- Seviye 1 (en üst)
        first_name as path                 -- Hiyerarşi yolu
    FROM employees
    WHERE manager_id IS NULL              -- Yöneticisi olmayan (CEO vb.)

    UNION ALL

    -- Recursive member: Tekrarlanan kısım (alt seviyeleri bul)
    SELECT
        e.employee_id,
        e.first_name,
        e.last_name,
        e.manager_id,
        eh.level_num + 1,                  -- Seviyeyi arttır
        eh.path || ' -> ' || e.first_name  -- Yolu genişlet
    FROM employees e
    JOIN emp_hierarchy eh ON e.manager_id = eh.employee_id  -- Üst seviyeyle bağla
)
SELECT
    level_num,
    LPAD(' ', (level_num-1)*2) || first_name as indented_name,  -- Girintili gösterim
    path,
    employee_id,
    manager_id
FROM emp_hierarchy
ORDER BY level_num, path;

-- Non-recursive CTE (normal kullanım)
-- Karmaşık sorguyu parçalara ayırma
WITH high_earners AS (
    -- 1. Adım: Yüksek maaşlı çalışanları bul
    SELECT employee_id, first_name, last_name, salary, department_id
    FROM employees
    WHERE salary > 8000
),
dept_stats AS (
    -- 2. Adım: Bu çalışanların departman istatistikleri
    SELECT
        department_id,
        AVG(salary) as avg_sal,
        COUNT(*) as emp_count,
        MAX(salary) as max_sal
    FROM high_earners
    GROUP BY department_id
),
top_departments AS (
    -- 3. Adım: En iyi departmanları seç
    SELECT department_id
    FROM dept_stats
    WHERE avg_sal > 10000 AND emp_count >= 2
)
-- Final sorgu: Tüm CTE'leri birleştir
SELECT
    he.first_name,
    he.last_name,
    he.salary,
    ds.avg_sal as dept_avg,
    ds.emp_count as dept_size
FROM high_earners he
JOIN dept_stats ds ON he.department_id = ds.department_id
JOIN top_departments td ON he.department_id = td.department_id
ORDER BY he.salary DESC;

-- Multiple CTE usage
WITH
salary_grades AS (
    SELECT
        employee_id,
        CASE
            WHEN salary > 15000 THEN 'A'
            WHEN salary > 10000 THEN 'B'
            WHEN salary > 5000 THEN 'C'
            ELSE 'D'
        END as grade
    FROM employees
),
grade_counts AS (
    SELECT grade, COUNT(*) as count
    FROM salary_grades
    GROUP BY grade
)
SELECT * FROM grade_counts ORDER BY grade;
```

## 9. Set Operations (Küme İşlemleri)

**Ne İşe Yarar:** İki veya daha fazla SELECT sorgusunun sonuçlarını birleştirir, ortak kısımları bulur veya farklarını alır.

```sql
-- UNION: Sonuçları birleştirir, tekrarları kaldırır
-- İki sorgu aynı sayıda sütun dönmeli ve tiplerinin uyuşması gerekir
SELECT first_name, last_name, 'Employee' as type
FROM employees
WHERE department_id = 10
UNION
SELECT first_name, last_name, 'High Earner' as type
FROM employees
WHERE salary > 10000;

-- UNION ALL: Sonuçları birleştirir, tekrarları saklar (daha hızlı)
-- Tekrar kontrolü yapmazmır, performans için tercih edilir
SELECT department_id, 'High Salary' as category
FROM employees
WHERE salary > 5000
UNION ALL
SELECT department_id, 'Recent Hire' as category
FROM employees
WHERE hire_date > DATE '2020-01-01';

-- INTERSECT: Her iki sorguda da bulunan ortak kayıtlar
-- Hem 10 numaralı departmanda HEM DE maaşı 5000'den fazla olan çalışanlar
SELECT employee_id, first_name, last_name
FROM employees
WHERE department_id = 10
INTERSECT
SELECT employee_id, first_name, last_name
FROM employees
WHERE salary > 5000;

-- MINUS (Oracle) / EXCEPT (SQL Standard): İlk sorguda var, ikincide yok
-- Tüm çalışanlar MINUS yüksek maaşlılar = düşük maaşlı çalışanlar
SELECT employee_id, first_name, last_name, salary
FROM employees
MINUS
SELECT employee_id, first_name, last_name, salary
FROM employees
WHERE salary > 10000;

-- Practical örnekler

-- Örnek 1: Farklı şehirlerdeki departmanları birleştir
SELECT d.department_name, l.city
FROM departments d
JOIN locations l ON d.location_id = l.location_id
WHERE l.city = 'Seattle'
UNION
SELECT d.department_name, l.city
FROM departments d
JOIN locations l ON d.location_id = l.location_id
WHERE l.city = 'Toronto';

-- Örnek 2: Aktif ve pasif çalışanları raporla
SELECT employee_id, first_name, last_name, 'Active' as status
FROM employees
WHERE department_id IS NOT NULL
UNION ALL
SELECT employee_id, first_name, last_name, 'No Department' as status
FROM employees
WHERE department_id IS NULL;

-- Örnek 3: Yüksek performanslı çalışanlar (maaş VEYA komisyon bazında)
SELECT employee_id, first_name, salary as amount, 'High Salary' as reason
FROM employees
WHERE salary > 12000
UNION
SELECT employee_id, first_name, salary * commission_pct as amount, 'High Commission' as reason
FROM employees
WHERE commission_pct > 0.3;

-- Set operations ile veri kalitesi kontrolü
-- Hem employees hem de temp_employees'da bulunan duplikasyonları bul
SELECT email
FROM employees
INTERSECT
SELECT email
FROM temp_employees;
```

## 10. Transaction Control (Transaction Yönetimi)

**Ne İşe Yarar:** Veritabanı değişikliklerini kontrollü bir şekilde yönetmek için kullanılır.

```sql
-- Transaction başlatma ve yönetimi
-- Not: Oracle'da ilk DML komutu otomatik transaction başlatır
BEGIN
    -- INSERT: Yeni kayıt ekleme
    INSERT INTO employees (employee_id, first_name, last_name, email, hire_date)
    VALUES (1001, 'Test', 'User', 'test@company.com', SYSDATE);

    -- İlk işlemden sonra kaç satır etkilendi kontrol et
    DBMS_OUTPUT.PUT_LINE('Inserted rows: ' || SQL%ROWCOUNT);

    -- UPDATE: Mevcut kayıtları güncelleme
    UPDATE employees
    SET salary = salary * 1.1
    WHERE department_id = 10;

    DBMS_OUTPUT.PUT_LINE('Updated rows: ' || SQL%ROWCOUNT);

    -- SAVEPOINT: Geçici kaydetme noktası oluştur
    -- Bu noktaya geri dönülebilir
    SAVEPOINT before_delete;

    -- DELETE: Kayıt silme
    DELETE FROM employees
    WHERE employee_id = 999;

    DBMS_OUTPUT.PUT_LINE('Deleted rows: ' || SQL%ROWCOUNT);

    -- Koşullu geri alma
    IF SQL%ROWCOUNT = 0 THEN
        -- Eğer silinecek kayıt bulunamadıysa sadece delete'i geri al
        ROLLBACK TO before_delete;
        DBMS_OUTPUT.PUT_LINE('Delete rolled back - no matching record');
    END IF;

    -- Tüm değişiklikleri kaydet
    COMMIT;
    DBMS_OUTPUT.PUT_LINE('Transaction committed successfully');

EXCEPTION
    WHEN OTHERS THEN
        -- Hata durumunda tüm değişiklikleri geri al
        ROLLBACK;
        DBMS_OUTPUT.PUT_LINE('Error occurred, transaction rolled back: ' || SQLERRM);
END;
/

-- Complex transaction örneği
-- Müşteri siparişi ve stok güncellemesi
DECLARE
    v_order_id NUMBER := 1001;
    v_product_id NUMBER := 505;
    v_quantity NUMBER := 5;
    v_current_stock NUMBER;
    insufficient_stock EXCEPTION;
BEGIN
    -- 1. Mevcut stok kontrolü
    SELECT stock_quantity INTO v_current_stock
    FROM products
    WHERE product_id = v_product_id;

    -- 2. Stok yeterli mi kontrol et
    IF v_current_stock < v_quantity THEN
        RAISE insufficient_stock;
    END IF;

    -- 3. Sipariş oluştur
    INSERT INTO orders (order_id, order_date, status)
    VALUES (v_order_id, SYSDATE, 'PENDING');

    -- 4. Sipariş detayı ekle
    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    SELECT v_order_id, v_product_id, v_quantity, unit_price
    FROM products
    WHERE product_id = v_product_id;

    -- 5. Stok güncelle
    UPDATE products
    SET stock_quantity = stock_quantity - v_quantity,
        last_updated = SYSDATE
    WHERE product_id = v_product_id;

    -- 6. Tüm işlemler başarılıysa kaydet
    COMMIT;
    DBMS_OUTPUT.PUT_LINE('Order ' || v_order_id || ' created successfully');

EXCEPTION
    WHEN insufficient_stock THEN
        ROLLBACK;
        DBMS_OUTPUT.PUT_LINE('Error: Insufficient stock. Available: ' || v_current_stock);
    WHEN NO_DATA_FOUND THEN
        ROLLBACK;
        DBMS_OUTPUT.PUT_LINE('Error: Product not found');
    WHEN OTHERS THEN
        ROLLBACK;
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
END;
/

-- Transaction isolation levels (session bazında)
-- READ COMMITTED (Oracle default)
ALTER SESSION SET ISOLATION_LEVEL = READ_COMMITTED;

-- SERIALIZABLE (en yüksek izolasyon)
ALTER SESSION SET ISOLATION_LEVEL = SERIALIZABLE;

-- Automatic transaction management
-- DDL komutları (CREATE, ALTER, DROP) otomatik COMMIT yapar
CREATE TABLE temp_table (id NUMBER);
-- Üsteki komut otomatik olarak önceki tüm değişiklikleri COMMIT eder

-- AUTOCOMMIT setting (SQL*Plus'ta)
-- SET AUTOCOMMIT ON;  -- Her DML sonrası otomatik COMMIT
-- SET AUTOCOMMIT OFF; -- Manuel COMMIT gerekir (default)
```

## 11. Views (Sanallar)

**Ne İşe Yarar:** View, bir veya daha fazla tabloya dayalı sanal tablolardır. Karmaşık sorguları basitleştirir ve güvenlik sağlar.

```sql
-- Basit view oluşturma
-- Çalışan ve departman bilgilerini birleştiren view
CREATE VIEW emp_dept_view AS
SELECT
    e.employee_id,
    e.first_name,
    e.last_name,
    e.salary,
    e.hire_date,
    d.department_name,
    d.location_id,
    l.city,
    l.country_id
FROM employees e
JOIN departments d ON e.department_id = d.department_id
JOIN locations l ON d.location_id = l.location_id;

-- View kullanımı (tıpkı tablo gibi)
SELECT * FROM emp_dept_view WHERE city = 'Seattle';
SELECT department_name, COUNT(*) FROM emp_dept_view GROUP BY department_name;

-- Complex view with calculations
-- Departman bazında maaş analizi view'u
CREATE VIEW salary_analysis AS
SELECT
    d.department_id,
    d.department_name,
    COUNT(e.employee_id) as emp_count,
    AVG(e.salary) as avg_salary,
    MAX(e.salary) as max_salary,
    MIN(e.salary) as min_salary,
    STDDEV(e.salary) as salary_stddev,
    SUM(e.salary) as total_salary_cost,
    -- Maaş dağılımı
    COUNT(CASE WHEN e.salary > 15000 THEN 1 END) as high_earners,
    COUNT(CASE WHEN e.salary BETWEEN 5000 AND 15000 THEN 1 END) as mid_earners,
    COUNT(CASE WHEN e.salary < 5000 THEN 1 END) as low_earners
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name;

-- Updatable view (güncellenebilir view)
-- Tek tablodan oluşan view'lar genelde güncellenebilir
CREATE VIEW active_employees AS
SELECT employee_id, first_name, last_name, salary, department_id
FROM employees
WHERE department_id IS NOT NULL
WITH CHECK OPTION;  -- UPDATE/INSERT'te WHERE koşulunu zorlar

-- View üzerinden güncelleme
UPDATE active_employees
SET salary = salary * 1.1
WHERE employee_id = 100;

-- Security view örneği
-- Hassas bilgileri gizleyen view
CREATE VIEW public_employee_info AS
SELECT
    employee_id,
    first_name,
    last_name,
    CASE
        WHEN LENGTH(email) > 0 THEN
            SUBSTR(email, 1, 2) || '***@' || SUBSTR(email, INSTR(email, '@')+1)
        ELSE NULL
    END as masked_email,  -- Email maskeleme
    department_id,
    hire_date,
    -- Maaş aralığı gösterme (tam sayı değil)
    CASE
        WHEN salary > 15000 THEN '15000+'
        WHEN salary > 10000 THEN '10000-15000'
        WHEN salary > 5000 THEN '5000-10000'
        ELSE '0-5000'
    END as salary_range
FROM employees;

-- Materialized view (performans için)
-- Fiziksel olarak sonuçları saklar, büyük veriler için hızlıdır
CREATE MATERIALIZED VIEW mv_dept_summary
REFRESH FAST ON COMMIT  -- Her COMMIT'te yenile
AS
SELECT
    department_id,
    COUNT(*) as emp_count,
    SUM(salary) as total_salary,
    AVG(salary) as avg_salary,
    MAX(hire_date) as last_hire_date
FROM employees
GROUP BY department_id;

-- Materialized view yenileme seçenekleri
-- REFRESH COMPLETE: Tümünü yeniden hesapla
-- REFRESH FAST: Sadece değişen kısımları güncelle (materialized view log gerekir)
-- REFRESH FORCE: Mümkünse FAST, değilse COMPLETE

-- Manual materialized view refresh
EXEC DBMS_MVIEW.REFRESH('mv_dept_summary', 'C');  -- Complete refresh

-- View metadata sorguları
-- Kullanıcının view'larını listele
SELECT view_name, text_length, read_only
FROM user_views;

-- View tanımını görüntüle
SELECT text FROM user_views WHERE view_name = 'EMP_DEPT_VIEW';

-- View silme
DROP VIEW emp_dept_view;
DROP MATERIALIZED VIEW mv_dept_summary;
```

## 12. Sequences (Sıra Numarası Üreticileri)

**Ne İşe Yarar:** Sequence, benzersiz sayısal değerler üreten Oracle nesnesidir. Genelde PRIMARY KEY için kullanılır.

```sql
-- Sequence oluşturma
CREATE SEQUENCE emp_seq
    START WITH 1000        -- Başlangıç değeri
    INCREMENT BY 1         -- Her seferinde artış miktarı
    MAXVALUE 999999        -- Maksimum değer
    MINVALUE 1             -- Minimum değer (CYCLE kullanıldığında)
    NOCYCLE                -- Max'a ulaştığında başa dönmesin
    CACHE 20               -- Performans için 20 değer önceden hafızaya al
    ORDER;                 -- Sıralı üretim garanti et (RAC ortamında önemli)

-- Sequence kullanımı
-- NEXTVAL: Sonraki değeri al ve sequence'i arttır
INSERT INTO employees (employee_id, first_name, last_name, email, hire_date)
VALUES (emp_seq.NEXTVAL, 'John', 'Doe', 'john.doe@company.com', SYSDATE);

-- CURRVAL: Son alınan değeri görüntüle (aynı session'da NEXTVAL'dan sonra kullanılabilir)
SELECT emp_seq.CURRVAL FROM DUAL;

-- Sequence bilgilerini görüntüleme
SELECT
    sequence_name,
    min_value,
    max_value,
    increment_by,
    last_number,           -- Son üretilen numara
    cache_size,
    cycle_flag,
    order_flag
FROM user_sequences
WHERE sequence_name = 'EMP_SEQ';

-- Sequence değiştirme
ALTER SEQUENCE emp_seq
    INCREMENT BY 5         -- Artış miktarını değiştir
    MAXVALUE 9999999       -- Max değeri yükselt
    CACHE 50;              -- Cache boyutunu artır

-- Sequence'i belirli bir değere set etme (doğrudan yok, trick gerekir)
-- Örneğin 5000'den başlamasını istiyorsak:
DECLARE
    current_val NUMBER;
    target_val NUMBER := 5000;
BEGIN
    -- Şu anki değeri al
    SELECT emp_seq.NEXTVAL INTO current_val FROM DUAL;

    -- Hedef değere kadar sequence'i arttır
    IF current_val < target_val THEN
        EXECUTE IMMEDIATE 'ALTER SEQUENCE emp_seq INCREMENT BY ' || (target_val - current_val);
        SELECT emp_seq.NEXTVAL INTO current_val FROM DUAL;  -- Hedef değere çıkar
        EXECUTE IMMEDIATE 'ALTER SEQUENCE emp_seq INCREMENT BY 1';  -- Normal artışa dön
    END IF;
END;
/

-- Farklı sequence örnekleri

-- Çift sayılar için sequence
CREATE SEQUENCE even_numbers_seq
    START WITH 2
    INCREMENT BY 2
    MAXVALUE 999998
    NOCYCLE
    CACHE 10;

-- Negatif yönde giden sequence
CREATE SEQUENCE countdown_seq
    START WITH 1000
    INCREMENT BY -1
    MINVALUE 1
    MAXVALUE 1000
    NOCYCLE
    CACHE 5;

-- Cycle yapan sequence (max'a ulaşınca başa döner)
CREATE SEQUENCE monthly_cycle_seq
    START WITH 1
    INCREMENT BY 1
    MAXVALUE 12
    MINVALUE 1
    CYCLE               -- 12'ye ulaşınca 1'e döner
    CACHE 3;

-- Practical kullanım örnekleri

-- Order numarası için
CREATE SEQUENCE order_number_seq
    START WITH 100001
    INCREMENT BY 1
    MAXVALUE 999999999
    NOCYCLE
    CACHE 100;          -- Yüksek cache (sık kullanım için)

-- Yıllık rapor ID'si için (2024001, 2024002, ...)
CREATE SEQUENCE report_2024_seq
    START WITH 2024001
    INCREMENT BY 1
    MAXVALUE 2024999
    NOCYCLE
    NOCACHE;            -- Cache yok (düşük kullanım)

-- Multiple table'da aynı sequence kullanma
INSERT INTO customers (customer_id, name) VALUES (emp_seq.NEXTVAL, 'ABC Corp');
INSERT INTO suppliers (supplier_id, name) VALUES (emp_seq.NEXTVAL, 'XYZ Ltd');
-- Her ikisi de farklı ID alacak

-- Sequence gaps (boşluklar)
-- ROLLBACK yapılırsa sequence geri almaz, boşluk oluşur
BEGIN
    INSERT INTO employees (employee_id, first_name) VALUES (emp_seq.NEXTVAL, 'Test');
    ROLLBACK;  -- INSERT geri alınır ama sequence numara boşa gider
END;
/

-- Sequence silme
DROP SEQUENCE emp_seq;

-- Performance ipucu: NOCACHE vs CACHE
-- NOCACHE: Her NEXTVAL için disk'e yazma (yavaş ama boşluk yok)
-- CACHE: Hafızada tutma (hızlı ama sistem crash'te boşluk olabilir)
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
