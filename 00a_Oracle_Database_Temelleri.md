# Oracle Database Temelleri

## 1. Oracle Database Nedir?

Oracle Database, enterprise seviyesinde ilişkisel veritabanı yönetim sistemidir (RDBMS). ACID özelliklerini destekler ve büyük ölçekli uygulamalar için tasarlanmıştır.

**Ana Özellikler:**

- **Scalability**: Büyük veri hacimleri
- **High Availability**: 7/24 çalışabilme
- **Security**: Güçlü güvenlik özellikleri
- **Performance**: Yüksek performans
- **Reliability**: Güvenilir veri yönetimi

## 2. Oracle Architecture

### Instance vs Database

**Oracle Instance:**

- Memory structures (SGA - System Global Area)
- Background processes
- Server'da çalışan Oracle software

**Oracle Database:**

- Physical files (datafiles, control files, redo logs)
- Disk üzerindeki fiziksel yapı

```
Oracle Instance - Hafıza ve Süreç Yapısı
├── SGA (System Global Area) - Paylaşılan Hafıza
│   ├── Database Buffer Cache - Veri sayfalarını saklar
│   ├── Shared Pool - SQL sorgu planları ve metadata
│   ├── Redo Log Buffer - Değişiklik kayıtları
│   └── Large Pool - Büyük işlemler için hafıza
├── PGA (Program Global Area) - Kullanıcı oturumu hafızası
└── Background Processes - Arka plan işlemleri
    ├── PMON (Process Monitor) - Süreç yönetimi
    ├── SMON (System Monitor) - Sistem temizliği
    ├── DBWn (Database Writer) - Veriyi diske yazar
    ├── LGWR (Log Writer) - Log dosyalarını yazar
    └── CKPT (Checkpoint) - Veri tutarlılığını sağlar
```

### Physical Database Structure

```sql
-- Database files görüntüleme
-- V$DATAFILE: Veri dosyalarının bilgilerini içeren view
-- NAME: Dosya yolu, BYTES: Dosya boyutu (byte cinsinden)
SELECT
    name,                        -- Veri dosyasının tam yolu
    bytes/1024/1024 as size_mb,  -- Boyutu MB cinsinden
    status,                      -- Dosya durumu (ONLINE, OFFLINE)
    enabled                      -- Dosya etkin mi?
FROM v$datafile;

-- V$LOGFILE: Redo log dosyalarının bilgileri
-- MEMBER: Log dosya yolu, GROUP#: Log grubu numarası
SELECT
    group#,                      -- Redo log grup numarası
    member,                      -- Log dosyasının tam yolu
    status                       -- Dosya durumu
FROM v$logfile
ORDER BY group#;

-- V$CONTROLFILE: Kontrol dosyalarının bilgileri
-- Kontrol dosyaları veritabanının metadata'larını saklar
SELECT
    name,                        -- Kontrol dosyasının tam yolu
    status                       -- Dosya durumu
FROM v$controlfile;

-- Temp dosyaları bilgileri
SELECT
    name,                        -- Temp dosya yolu
    bytes/1024/1024 as size_mb,  -- Boyut MB cinsinden
    status                       -- Dosya durumu
FROM v$tempfile;
```

## 3. Logical Database Structure

### Tablespace

Veritabanının logical storage unit'i:

```sql
-- Tablespace oluşturma
-- CREATE TABLESPACE: Yeni bir logical storage alanı oluşturur
-- DATAFILE: Fiziksel dosya yolunu belirtir
-- SIZE: Başlangıç boyutu
-- AUTOEXTEND ON: Otomatik büyüme özelliği
-- NEXT: Her seferinde ne kadar büyüyeceği
-- MAXSIZE: Maksimum boyut sınırı
CREATE TABLESPACE app_data
DATAFILE '/u01/app/oracle/oradata/ORCL/app_data01.dbf'
SIZE 100M                    -- Başlangıçta 100 MB
AUTOEXTEND ON NEXT 10M       -- Dolduğunda 10 MB artır
MAXSIZE 1G;                  -- En fazla 1 GB olabilir

-- Tablespace'e ek datafile ekleme
ALTER TABLESPACE app_data
ADD DATAFILE '/u01/app/oracle/oradata/ORCL/app_data02.dbf'
SIZE 200M AUTOEXTEND ON;

-- DBA_DATA_FILES: Tüm veri dosyalarının bilgilerini içerir
-- TABLESPACE_NAME: Tablespace adı, BYTES: Dosya boyutu
SELECT
    tablespace_name,             -- Tablespace adı
    file_name,                   -- Dosya tam yolu
    bytes/1024/1024 as size_mb,  -- Dosya boyutu MB
    status,                      -- Dosya durumu (AVAILABLE, INVALID)
    autoextensible               -- Otomatik genişletme var mı?
FROM dba_data_files
ORDER BY tablespace_name;

-- Tablespace kullanım analizi - karşılaştırmalı rapor
-- Bu sorgu her tablespace'in toplam, kullanılan ve boş alanını gösterir
SELECT
    t.tablespace_name,
    t.total_mb,                  -- Toplam alan
    f.free_mb,                   -- Boş alan
    t.total_mb - f.free_mb as used_mb,  -- Kullanılan alan
    ROUND(((t.total_mb - f.free_mb)/t.total_mb)*100, 2) as used_percent
FROM (
    -- Toplam alan hesaplama
    SELECT tablespace_name, SUM(bytes)/1024/1024 as total_mb
    FROM dba_data_files
    GROUP BY tablespace_name
) t
JOIN (
    -- Boş alan hesaplama (DBA_FREE_SPACE: Boş alanları gösterir)
    SELECT tablespace_name, SUM(bytes)/1024/1024 as free_mb
    FROM dba_free_space
    GROUP BY tablespace_name
) f ON t.tablespace_name = f.tablespace_name
ORDER BY used_percent DESC;

-- Tablespace offline/online yapma
ALTER TABLESPACE app_data OFFLINE;  -- Bakım için offline yap
ALTER TABLESPACE app_data ONLINE;   -- Tekrar online yap

-- Tablespace silme (DIİKKAT: Veriyi siler!)
DROP TABLESPACE app_data INCLUDING CONTENTS AND DATAFILES;
```

### Schema

Bir kullanıcıya ait olan database object'lerin koleksiyonu:

```sql
-- Schema (user) oluşturma
-- CREATE USER: Yeni veritabanı kullanıcısı oluşturur
-- IDENTIFIED BY: Şifre belirler
-- DEFAULT TABLESPACE: Varsayılan tablespace (kullanıcının nesneleri burada saklanır)
-- TEMPORARY TABLESPACE: Geçici işlemler için tablespace
-- QUOTA: Belirtilen tablespace'de ne kadar alan kullanabileceği
CREATE USER hr_user
IDENTIFIED BY password123      -- Kullanıcı şifresi
DEFAULT TABLESPACE app_data    -- Varsayılan tablespace
TEMPORARY TABLESPACE temp      -- Geçici işlemler için
QUOTA 100M ON app_data;        -- app_data'da 100MB kullanabilir

-- Ek tablespace quota'ları verme
ALTER USER hr_user QUOTA 50M ON users;
ALTER USER hr_user QUOTA UNLIMITED ON app_data;  -- Sınırsız alan

-- Yetki verme (System Privileges)
-- CONNECT: Veritabanına bağlanma yetkisi
-- RESOURCE: Tablo, index vb. nesneler oluşturma yetkisi
-- CREATE VIEW: View oluşturma yetkisi
-- CREATE PROCEDURE: Stored procedure oluşturma yetkisi
GRANT CONNECT, RESOURCE TO hr_user;
GRANT CREATE VIEW, CREATE PROCEDURE TO hr_user;
GRANT CREATE SEQUENCE, CREATE TRIGGER TO hr_user;

-- Object privileges (belirli nesneler üzerinde yetkiler)
GRANT SELECT, INSERT, UPDATE ON employees TO hr_user;
GRANT ALL ON departments TO hr_user;  -- Tüm yetkiler

-- DBA_USERS: Tüm kullanıcıların bilgilerini gösterir
SELECT
    username,                    -- Kullanıcı adı
    default_tablespace,          -- Varsayılan tablespace
    temporary_tablespace,        -- Geçici tablespace
    created,                     -- Oluşturulma tarihi
    account_status,              -- Hesap durumu (OPEN, LOCKED, EXPIRED)
    profile,                     -- Güvenlik profili
    authentication_type          -- Kimlik doğrulama tipi
FROM dba_users
WHERE username = 'HR_USER';

-- Kullanıcı quota'larını görme
SELECT
    username,
    tablespace_name,
    bytes/1024/1024 as used_mb,  -- Kullanılan alan
    max_bytes/1024/1024 as quota_mb  -- Verilen quota (-1 = UNLIMITED)
FROM dba_ts_quotas
WHERE username = 'HR_USER';

-- Kullanıcı silme
DROP USER hr_user CASCADE;       -- CASCADE: Kullanıcının nesnelerini de siler
```

## 4. Oracle Data Types

### Karakter Veri Tipleri

```sql
CREATE TABLE data_types_demo (
    -- Karakter tipleri
    fixed_char CHAR(10),           -- Sabit uzunluk
    variable_char VARCHAR2(100),   -- Değişken uzunluk (önerilen)
    unicode_text NVARCHAR2(50),   -- Unicode destekli
    large_text CLOB,              -- 4GB'a kadar text
    unicode_large NCLOB           -- Unicode CLOB
);
```

### Numeric Veri Tipleri

```sql
CREATE TABLE numeric_demo (
    exact_number NUMBER(10,2),     -- Tam hassasiyet
    integer_val INTEGER,           -- Tam sayı
    float_val FLOAT(10),          -- Yaklaşık sayı
    binary_float BINARY_FLOAT,     -- 32-bit float
    binary_double BINARY_DOUBLE    -- 64-bit double
);
```

### Date ve Time Tipleri

```sql
CREATE TABLE datetime_demo (
    simple_date DATE,                    -- Tarih ve saat
    precise_time TIMESTAMP,              -- Mikrosaniye hassasiyeti
    with_timezone TIMESTAMP WITH TIME ZONE,
    local_timezone TIMESTAMP WITH LOCAL TIME ZONE,
    year_month INTERVAL YEAR TO MONTH,
    day_second INTERVAL DAY TO SECOND
);

-- Örnek kullanım
INSERT INTO datetime_demo VALUES (
    SYSDATE,
    SYSTIMESTAMP,
    TIMESTAMP '2024-01-15 14:30:00.123456 +03:00',
    TIMESTAMP '2024-01-15 14:30:00.123456',
    INTERVAL '2-6' YEAR TO MONTH,      -- 2 yıl 6 ay
    INTERVAL '5 10:30:15' DAY TO SECOND -- 5 gün 10 saat 30 dakika 15 saniye
);
```

### LOB (Large Object) Tipleri

```sql
CREATE TABLE lob_demo (
    id NUMBER,
    text_data CLOB,        -- Karakter LOB
    binary_data BLOB,      -- Binary LOB
    file_pointer BFILE     -- External file pointer
);
```

## 5. Constraints (Kısıtlamalar)

### Primary Key

```sql
-- Tablo oluştururken Primary Key tanımlama
-- PRIMARY KEY: Her satırı benzersiz tanımlayan birincil anahtar
-- Otomatik olarak UNIQUE ve NOT NULL constraint'i de ekler
-- Otomatik olarak bir UNIQUE INDEX oluşturur
CREATE TABLE departments (
    dept_id NUMBER PRIMARY KEY,  -- Inline tanımlama
    dept_name VARCHAR2(50) NOT NULL
);

-- İsimlendirilmiş constraint ile tanımlama (tercih edilen yöntem)
CREATE TABLE employees (
    emp_id NUMBER,
    first_name VARCHAR2(50),
    -- CONSTRAINT: Constraint'e isim verir (hata mesajlarında kullanılır)
    CONSTRAINT pk_emp PRIMARY KEY (emp_id)
);

-- Composite Primary Key (birden fazla sütunlu)
CREATE TABLE order_items (
    order_id NUMBER,
    item_id NUMBER,
    quantity NUMBER,
    CONSTRAINT pk_order_items PRIMARY KEY (order_id, item_id)
);

-- Mevcut tabloya Primary Key ekleme
ALTER TABLE employees
ADD CONSTRAINT pk_emp PRIMARY KEY (emp_id);

-- Primary Key constraint'ini silme
ALTER TABLE employees DROP CONSTRAINT pk_emp;
```

### Foreign Key

```sql
CREATE TABLE employees (
    emp_id NUMBER PRIMARY KEY,
    first_name VARCHAR2(50),
    dept_id NUMBER,
    CONSTRAINT fk_emp_dept FOREIGN KEY (dept_id)
        REFERENCES departments(dept_id)
);

-- Cascade options
ALTER TABLE employees
ADD CONSTRAINT fk_emp_dept FOREIGN KEY (dept_id)
REFERENCES departments(dept_id) ON DELETE CASCADE;
```

### Check Constraints

```sql
CREATE TABLE employees (
    emp_id NUMBER PRIMARY KEY,
    salary NUMBER CONSTRAINT chk_salary CHECK (salary > 0),
    email VARCHAR2(100) CONSTRAINT chk_email
        CHECK (email LIKE '%@%.%'),
    status VARCHAR2(10) CONSTRAINT chk_status
        CHECK (status IN ('ACTIVE', 'INACTIVE', 'TERMINATED'))
);
```

### Unique Constraints

```sql
CREATE TABLE employees (
    emp_id NUMBER PRIMARY KEY,
    email VARCHAR2(100) UNIQUE,
    ssn VARCHAR2(11) CONSTRAINT uk_ssn UNIQUE
);
```

## 6. Indexes

### B-Tree Index (Default)

```sql
-- Single column index
-- CREATE INDEX: Sorgu performansını artırmak için index oluşturur
-- B-Tree: Oracle'da varsayılan index tipi (dengeli ağaç yapısı)
CREATE INDEX idx_emp_lastname ON employees(last_name);

-- Composite index (birden fazla sütunlu)
-- Sütun sırası önemli: en seçici sütun başta olmalı
CREATE INDEX idx_emp_dept_sal ON employees(department_id, salary);

-- Unique index (benzersiz değerler için)
-- Aynı zamanda UNIQUE constraint sağlar
CREATE UNIQUE INDEX idx_emp_email ON employees(email);

-- Descending index (azalan sıralama için optimize)
CREATE INDEX idx_emp_sal_desc ON employees(salary DESC);

-- Index bilgilerini görüntüleme
-- USER_INDEXES: Kullanıcının sahip olduğu index'ler
SELECT
    index_name,                  -- Index adı
    table_name,                  -- Hangi tabloya ait
    uniqueness,                  -- UNIQUE mi NONUNIQUE mi
    status,                      -- Durumu (VALID, INVALID)
    degree,                      -- Parallellik derecesi
    last_analyzed                -- Son analiz tarihi
FROM user_indexes
WHERE table_name = 'EMPLOYEES';

-- Index sütunlarını görme
-- USER_IND_COLUMNS: Index'lerin hangi sütunları içerdiği
SELECT
    index_name,
    column_name,
    column_position,             -- Index içindeki sırası
    descend                      -- ASC mi DESC mi
FROM user_ind_columns
WHERE table_name = 'EMPLOYEES'
ORDER BY index_name, column_position;

-- Index kullanım istatistikleri (11g+)
SELECT
    name,                        -- Index adı
    total_access_count,          -- Toplam erişim sayısı
    total_exec_count,            -- Toplam çalıştırma sayısı
    bucket_0_access_count        -- Hiç kullanılmama sayısı
FROM v$index_usage_info
WHERE name LIKE 'IDX_EMP%';

-- Index yeniden oluşturma (fragmentation temizliği)
ALTER INDEX idx_emp_lastname REBUILD;
ALTER INDEX idx_emp_lastname REBUILD ONLINE;  -- Sistem çalışırken

-- Index silme
DROP INDEX idx_emp_lastname;
```

### Bitmap Index

```sql
-- Low cardinality columns için
CREATE BITMAP INDEX idx_emp_gender ON employees(gender);
CREATE BITMAP INDEX idx_emp_status ON employees(status);
```

### Function-Based Index

```sql
-- Function sonucu üzerinde index
CREATE INDEX idx_emp_upper_name ON employees(UPPER(last_name));
CREATE INDEX idx_emp_salary_tax ON employees(salary * 0.3);

-- Bu index kullanılacak
SELECT * FROM employees WHERE UPPER(last_name) = 'SMITH';
```

### Reverse Key Index

```sql
-- Sequential insert'ler için hot block sorununu çözer
CREATE INDEX idx_emp_id_rev ON employees(employee_id) REVERSE;
```

## 7. Partitioning

### Range Partitioning

```sql
CREATE TABLE sales (
    sale_id NUMBER,
    sale_date DATE,
    amount NUMBER
)
PARTITION BY RANGE (sale_date) (
    PARTITION sales_2023 VALUES LESS THAN (DATE '2024-01-01'),
    PARTITION sales_2024 VALUES LESS THAN (DATE '2025-01-01'),
    PARTITION sales_future VALUES LESS THAN (MAXVALUE)
);
```

### Hash Partitioning

```sql
CREATE TABLE employees (
    emp_id NUMBER,
    first_name VARCHAR2(50),
    last_name VARCHAR2(50)
)
PARTITION BY HASH (emp_id)
PARTITIONS 4;
```

### List Partitioning

```sql
CREATE TABLE sales_by_region (
    sale_id NUMBER,
    region VARCHAR2(20),
    amount NUMBER
)
PARTITION BY LIST (region) (
    PARTITION north VALUES ('NORTH', 'NORTHEAST'),
    PARTITION south VALUES ('SOUTH', 'SOUTHEAST'),
    PARTITION west VALUES ('WEST', 'NORTHWEST'),
    PARTITION east VALUES ('EAST')
);
```

## 8. Security

### Users ve Roles

```sql
-- Role oluşturma
CREATE ROLE app_user_role;
CREATE ROLE app_admin_role;

-- Role'lere yetki verme
GRANT SELECT, INSERT, UPDATE ON employees TO app_user_role;
GRANT ALL ON employees TO app_admin_role;

-- User'a role atama
GRANT app_user_role TO hr_user;
GRANT app_admin_role TO hr_admin;

-- Object privileges
GRANT SELECT ON employees TO public;
GRANT INSERT, UPDATE ON employees TO hr_user;
GRANT EXECUTE ON employee_pkg TO app_user_role;
```

### VPD (Virtual Private Database)

```sql
-- Security policy function
CREATE OR REPLACE FUNCTION emp_security_policy(
    schema_var IN VARCHAR2,
    table_var IN VARCHAR2
) RETURN VARCHAR2 IS
BEGIN
    IF USER = 'HR_ADMIN' THEN
        RETURN NULL; -- No restriction
    ELSE
        RETURN 'department_id = SYS_CONTEXT(''USERENV'', ''CLIENT_IDENTIFIER'')';
    END IF;
END;
/

-- Policy uygulama
BEGIN
    DBMS_RLS.ADD_POLICY(
        object_schema => 'HR',
        object_name => 'EMPLOYEES',
        policy_name => 'EMP_DEPT_POLICY',
        function_schema => 'HR',
        policy_function => 'EMP_SECURITY_POLICY',
        statement_types => 'SELECT, INSERT, UPDATE, DELETE'
    );
END;
/
```

## 9. Backup ve Recovery

### RMAN (Recovery Manager)

```sql
-- Full database backup
BACKUP DATABASE;

-- Incremental backup
BACKUP INCREMENTAL LEVEL 1 DATABASE;

-- Tablespace backup
BACKUP TABLESPACE users;

-- Archive log backup
BACKUP ARCHIVELOG ALL;

-- Point-in-time recovery
RECOVER DATABASE UNTIL TIME '2024-01-15 14:30:00';
```

### Export/Import

```bash
# Data Pump Export
expdp hr/password directory=dp_dir dumpfile=hr_backup.dmp schemas=hr

# Data Pump Import
impdp hr/password directory=dp_dir dumpfile=hr_backup.dmp schemas=hr

# Table level export
expdp hr/password directory=dp_dir dumpfile=emp_backup.dmp tables=employees
```

## 10. Performance Monitoring

### AWR (Automatic Workload Repository)

```sql
-- AWR snapshot oluşturma
-- AWR: Oracle'da performans izleme sistemi
-- Snapshot: Belirli bir andaki sistem durumunun fotografı
-- Manuel snapshot oluşturma (normalde otomatik alınır)
EXEC DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();

-- AWR snapshot'larını listeleme
SELECT
    snap_id,                     -- Snapshot ID'si
    begin_interval_time,         -- Başlangıç zamanı
    end_interval_time,           -- Bitiş zamanı
    snap_level                   -- Snapshot seviyesi (detayı)
FROM dba_hist_snapshot
WHERE begin_interval_time >= SYSDATE - 1  -- Son 24 saat
ORDER BY snap_id DESC;

-- AWR report oluşturma (SQL*Plus'da çalıştır)
@$ORACLE_HOME/rdbms/admin/awrrpt.sql

-- En yük CPU tüketen SQL'leri bulma
-- V$SQL: Şu anda SGA'da bulunan SQL ifadeleri
-- ELAPSED_TIME: Toplam geçen süre (mikrosaniye)
-- EXECUTIONS: Çalıştırma sayısı
SELECT
    sql_id,                      -- SQL'in benzersiz ID'si
    executions,                  -- Kaç kez çalıştırıldı
    elapsed_time/executions/1000000 as avg_etime_sec,  -- Ortalama süre (saniye)
    cpu_time/1000000 as cpu_sec, -- CPU süresi (saniye)
    buffer_gets,                 -- Buffer cache'den okunan blok sayısı
    disk_reads,                  -- Diskten okunan blok sayısı
    SUBSTR(sql_text, 1, 100) as sql_text  -- SQL metninin ilk 100 karakteri
FROM (
    SELECT
        sql_id, executions, elapsed_time, cpu_time,
        buffer_gets, disk_reads, sql_text,
        RANK() OVER (ORDER BY elapsed_time DESC) as rnk
    FROM v$sql
    WHERE executions > 0         -- En az bir kez çalıştırılmış
    AND sql_text NOT LIKE 'SELECT%FROM V$%'  -- System query'leri hariç tut
)
WHERE rnk <= 10                 -- En yük 10 SQL
ORDER BY elapsed_time DESC;

-- Session bazlı performans analizi
SELECT
    s.sid,                       -- Session ID
    s.serial#,                   -- Session Serial #
    s.username,                  -- Kullanıcı adı
    s.program,                   -- Çalıştıran program
    s.machine,                   -- Client makinesi
    s.status,                    -- Session durumu (ACTIVE, INACTIVE)
    s.last_call_et,              -- Son çağrıdan beri geçen süre
    sq.sql_text                  -- Şu an çalışan SQL
FROM v$session s
LEFT JOIN v$sql sq ON s.sql_id = sq.sql_id
WHERE s.username IS NOT NULL    -- Background process'leri hariç tut
AND s.status = 'ACTIVE'         -- Sadece aktif session'lar
ORDER BY s.last_call_et;

-- Wait events analizi (neyin beklendiğini görür)
SELECT
    event,                       -- Bekleme event'i
    total_waits,                 -- Toplam bekleme sayısı
    total_timeouts,              -- Timeout sayısı
    time_waited/100 as time_waited_sec,  -- Toplam bekleme süresi
    average_wait/100 as avg_wait_sec     -- Ortalama bekleme süresi
FROM v$system_event
WHERE wait_class != 'Idle'      -- Idle event'leri hariç tut
ORDER BY time_waited DESC;
```

### Execution Plans

```sql
-- Explain plan
EXPLAIN PLAN FOR
SELECT e.first_name, d.department_name
FROM employees e JOIN departments d ON e.department_id = d.department_id
WHERE e.salary > 5000;

-- Plan görüntüleme
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Real execution statistics
SELECT /*+ GATHER_PLAN_STATISTICS */
    e.first_name, d.department_name
FROM employees e JOIN departments d ON e.department_id = d.department_id
WHERE e.salary > 5000;

-- Actual plan
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST'));
```

## 11. Oracle-Specific Features

### Hierarchical Queries

```sql
-- CONNECT BY ile hierarchy
SELECT LEVEL, LPAD(' ', (LEVEL-1)*2) || first_name as hierarchy
FROM employees
START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id = manager_id;

-- Recursive CTE (12c+)
WITH emp_hierarchy (employee_id, first_name, manager_id, lvl) AS (
    SELECT employee_id, first_name, manager_id, 1
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.employee_id, e.first_name, e.manager_id, eh.lvl + 1
    FROM employees e
    JOIN emp_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM emp_hierarchy;
```

### Model Clause

```sql
-- Spreadsheet-like calculations
SELECT year, month, sales,
       sales_growth
FROM sales_data
MODEL
    PARTITION BY (product_id)
    DIMENSION BY (year, month)
    MEASURES (sales, 0 as sales_growth)
    RULES (
        sales_growth[ANY, ANY] =
            CASE WHEN sales[CV(year), CV(month)-1] IS NOT NULL
            THEN (sales[CV(year), CV(month)] - sales[CV(year), CV(month)-1]) /
                 sales[CV(year), CV(month)-1] * 100
            ELSE NULL END
    );
```

### Flashback Features

```sql
-- Flashback Query
SELECT * FROM employees
AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '1' HOUR)
WHERE employee_id = 100;

-- Flashback Table
FLASHBACK TABLE employees TO TIMESTAMP (SYSTIMESTAMP - INTERVAL '30' MINUTE);

-- Flashback Drop
FLASHBACK TABLE employees TO BEFORE DROP;
```

## 12. JSON Support (12c+)

```sql
-- JSON sütunu
CREATE TABLE products (
    id NUMBER,
    data JSON,
    CONSTRAINT chk_json CHECK (data IS JSON)
);

-- JSON veri ekleme
INSERT INTO products VALUES (1, '{
    "name": "Laptop",
    "brand": "Dell",
    "specs": {
        "cpu": "Intel i7",
        "ram": "16GB",
        "storage": "512GB SSD"
    },
    "tags": ["computer", "portable", "business"]
}');

-- JSON sorgulama
SELECT p.data.name, p.data.brand
FROM products p;

-- JSON_TABLE kullanımı
SELECT jt.*
FROM products p,
     JSON_TABLE(p.data, '$'
         COLUMNS (
             name VARCHAR2(100) PATH '$.name',
             brand VARCHAR2(50) PATH '$.brand',
             cpu VARCHAR2(50) PATH '$.specs.cpu',
             ram VARCHAR2(20) PATH '$.specs.ram'
         )
     ) jt;
```

## Database Best Practices

### Design Principles

1. **Normalization**: Veri tekrarını minimize edin
2. **Constraints**: Veri bütünlüğünü garanti edin
3. **Indexing**: Performance için doğru index'leri oluşturun
4. **Partitioning**: Büyük tablolar için partition kullanın

### Performance Tips

1. **Statistics**: Güncel istatistikleri tutun
2. **SQL Tuning**: Execution plan'ları analiz edin
3. **Memory Management**: SGA ve PGA'yı optimize edin
4. **I/O Distribution**: Datafile'ları farklı disk'lere dağıtın

### Security Best Practices

1. **Least Privilege**: Minimum gerekli yetkileri verin
2. **Audit**: Kritik operasyonları loglayin
3. **Encryption**: Sensitive data'yı şifreleyin
4. **Regular Updates**: Security patch'leri uygulayın

**Sonraki Bölümde:** PL/SQL temellerine geçeceğiz.
