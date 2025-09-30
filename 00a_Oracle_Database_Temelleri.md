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
Oracle Instance
├── SGA (System Global Area)
│   ├── Database Buffer Cache
│   ├── Shared Pool
│   ├── Redo Log Buffer
│   └── Large Pool
├── PGA (Program Global Area)
└── Background Processes
    ├── PMON (Process Monitor)
    ├── SMON (System Monitor)
    ├── DBWn (Database Writer)
    ├── LGWR (Log Writer)
    └── CKPT (Checkpoint)
```

### Physical Database Structure

```sql
-- Database files görüntüleme
SELECT name, bytes/1024/1024 as size_mb FROM v$datafile;
SELECT member FROM v$logfile;
SELECT name FROM v$controlfile;
```

## 3. Logical Database Structure

### Tablespace

Veritabanının logical storage unit'i:

```sql
-- Tablespace oluşturma
CREATE TABLESPACE app_data
DATAFILE '/u01/app/oracle/oradata/ORCL/app_data01.dbf'
SIZE 100M
AUTOEXTEND ON NEXT 10M
MAXSIZE 1G;

-- Tablespace bilgileri
SELECT tablespace_name, bytes/1024/1024 as size_mb, status
FROM dba_data_files;

-- Tablespace kullanımı
SELECT
    tablespace_name,
    total_mb,
    used_mb,
    free_mb,
    ROUND((used_mb/total_mb)*100, 2) as used_percent
FROM (
    SELECT
        tablespace_name,
        SUM(bytes)/1024/1024 as total_mb
    FROM dba_data_files
    GROUP BY tablespace_name
) t1
JOIN (
    SELECT
        tablespace_name,
        SUM(bytes)/1024/1024 as free_mb
    FROM dba_free_space
    GROUP BY tablespace_name
) t2 USING (tablespace_name),
(
    SELECT
        tablespace_name,
        total_mb - free_mb as used_mb
    FROM (
        SELECT tablespace_name, SUM(bytes)/1024/1024 as total_mb
        FROM dba_data_files GROUP BY tablespace_name
    ) t1
    JOIN (
        SELECT tablespace_name, SUM(bytes)/1024/1024 as free_mb
        FROM dba_free_space GROUP BY tablespace_name
    ) t2 USING (tablespace_name)
) t3 USING (tablespace_name);
```

### Schema

Bir kullanıcıya ait olan database object'lerin koleksiyonu:

```sql
-- Schema (user) oluşturma
CREATE USER hr_user
IDENTIFIED BY password123
DEFAULT TABLESPACE app_data
TEMPORARY TABLESPACE temp
QUOTA 100M ON app_data;

-- Yetki verme
GRANT CONNECT, RESOURCE TO hr_user;
GRANT CREATE VIEW, CREATE PROCEDURE TO hr_user;

-- Schema bilgileri
SELECT username, default_tablespace, temporary_tablespace, created
FROM dba_users
WHERE username = 'HR_USER';
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
CREATE TABLE departments (
    dept_id NUMBER PRIMARY KEY,
    dept_name VARCHAR2(50) NOT NULL
);

-- Alternative syntax
CREATE TABLE employees (
    emp_id NUMBER,
    first_name VARCHAR2(50),
    CONSTRAINT pk_emp PRIMARY KEY (emp_id)
);
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
CREATE INDEX idx_emp_lastname ON employees(last_name);

-- Composite index
CREATE INDEX idx_emp_dept_sal ON employees(department_id, salary);

-- Unique index
CREATE UNIQUE INDEX idx_emp_email ON employees(email);

-- Index bilgileri
SELECT index_name, table_name, uniqueness, status
FROM user_indexes
WHERE table_name = 'EMPLOYEES';
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
EXEC DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();

-- AWR report oluşturma
@$ORACLE_HOME/rdbms/admin/awrrpt.sql

-- Top SQL queries
SELECT sql_id, executions, avg_etime, sql_text
FROM (
    SELECT sql_id, executions,
           elapsed_time/executions/1000000 as avg_etime,
           sql_text,
           RANK() OVER (ORDER BY elapsed_time DESC) as rnk
    FROM v$sql
    WHERE executions > 0
)
WHERE rnk <= 10;
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
