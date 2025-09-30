# Procedure ve Function'lar

## 1. Procedure Nedir?

Procedure, belirli bir işlemi gerçekleştiren, tekrar kullanılabilir kod bloklarıdır. Değer döndürmez, işlem yapar.

**Neden Kullanılır?**

- Kod tekrarını önler
- Modüler programlama sağlar
- Merkezi iş mantığı yönetimi
- Performans artışı (compile edilmiş halde saklanır)
- Güvenlik (SQL injection korunması)

## 2. Procedure Oluşturma

### Basit Procedure

```sql
CREATE OR REPLACE PROCEDURE employee_info(p_emp_id IN NUMBER) IS
    v_first_name employees.first_name%TYPE;
    v_last_name employees.last_name%TYPE;
    v_salary employees.salary%TYPE;
    v_department departments.department_name%TYPE;
BEGIN
    SELECT e.first_name, e.last_name, e.salary, d.department_name
    INTO v_first_name, v_last_name, v_salary, v_department
    FROM employees e
    JOIN departments d ON e.department_id = d.department_id
    WHERE e.employee_id = p_emp_id;

    DBMS_OUTPUT.PUT_LINE('Çalışan: ' || v_first_name || ' ' || v_last_name);
    DBMS_OUTPUT.PUT_LINE('Maaş: ' || v_salary);
    DBMS_OUTPUT.PUT_LINE('Departman: ' || v_department);
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Çalışan bulunamadı: ' || p_emp_id);
END;
/

-- Çağırma
BEGIN
    employee_info(100);
END;
/
```

### Parametreli Procedure

```sql
CREATE OR REPLACE PROCEDURE salary_increase(
    p_emp_id IN NUMBER,
    p_increase_percent IN NUMBER,
    p_result OUT VARCHAR2
) IS
    v_current_salary NUMBER;
    v_new_salary NUMBER;
    v_emp_name VARCHAR2(100);
BEGIN
    -- Mevcut maaş bilgisini al
    SELECT salary, first_name || ' ' || last_name
    INTO v_current_salary, v_emp_name
    FROM employees
    WHERE employee_id = p_emp_id;

    -- Yeni maaşı hesapla
    v_new_salary := v_current_salary * (1 + p_increase_percent / 100);

    -- Güncelle
    UPDATE employees
    SET salary = v_new_salary
    WHERE employee_id = p_emp_id;

    p_result := v_emp_name || ' maaşı ' || v_current_salary ||
                ' den ' || v_new_salary || ' ye güncellendi.';

    COMMIT;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        p_result := 'Hata: Çalışan bulunamadı!';
        ROLLBACK;
    WHEN OTHERS THEN
        p_result := 'Hata: ' || SQLERRM;
        ROLLBACK;
END;
/

-- Çağırma
DECLARE
    v_message VARCHAR2(200);
BEGIN
    salary_increase(100, 10, v_message);
    DBMS_OUTPUT.PUT_LINE(v_message);
END;
/
```

## 3. Function Nedir?

Function, bir değer döndüren procedure'dür. SELECT sorgularında kullanılabilir.

### Basit Function

```sql
CREATE OR REPLACE FUNCTION get_employee_salary(p_emp_id NUMBER)
RETURN NUMBER IS
    v_salary employees.salary%TYPE;
BEGIN
    SELECT salary INTO v_salary
    FROM employees
    WHERE employee_id = p_emp_id;

    RETURN v_salary;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN NULL;
    WHEN OTHERS THEN
        RETURN -1; -- Hata durumu
END;
/

-- Kullanım
SELECT employee_id, first_name, get_employee_salary(employee_id) as salary
FROM employees
WHERE department_id = 10;
```

### Karmaşık Function Örneği

```sql
CREATE OR REPLACE FUNCTION calculate_bonus(
    p_emp_id NUMBER,
    p_performance_score NUMBER
) RETURN NUMBER IS
    v_salary NUMBER;
    v_job_id VARCHAR2(10);
    v_bonus NUMBER := 0;
    v_years_worked NUMBER;
BEGIN
    -- Çalışan bilgilerini al
    SELECT salary, job_id,
           TRUNC(MONTHS_BETWEEN(SYSDATE, hire_date) / 12) as years
    INTO v_salary, v_job_id, v_years_worked
    FROM employees
    WHERE employee_id = p_emp_id;

    -- Temel bonus hesaplama
    CASE
        WHEN p_performance_score >= 90 THEN
            v_bonus := v_salary * 0.15; -- %15 bonus
        WHEN p_performance_score >= 80 THEN
            v_bonus := v_salary * 0.10; -- %10 bonus
        WHEN p_performance_score >= 70 THEN
            v_bonus := v_salary * 0.05; -- %5 bonus
        ELSE
            v_bonus := 0;
    END CASE;

    -- Kıdem bonusu
    IF v_years_worked > 5 THEN
        v_bonus := v_bonus * 1.2; -- %20 artış
    END IF;

    -- Pozisyon bonusu
    IF v_job_id LIKE '%_MAN' OR v_job_id LIKE '%_MGR' THEN
        v_bonus := v_bonus * 1.1; -- %10 artış
    END IF;

    RETURN ROUND(v_bonus, 2);
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN 0;
    WHEN OTHERS THEN
        RETURN -1;
END;
/

-- Kullanım
SELECT employee_id,
       first_name || ' ' || last_name as full_name,
       salary,
       calculate_bonus(employee_id, 85) as bonus_amount
FROM employees
WHERE department_id = 50
ORDER BY bonus_amount DESC;
```

## 4. Parametre Modları

### IN Parametresi (Varsayılan)

```sql
CREATE OR REPLACE PROCEDURE demo_in(p_value IN NUMBER) IS
BEGIN
    DBMS_OUTPUT.PUT_LINE('Gelen değer: ' || p_value);
    -- p_value := 100; -- HATA! IN parametresi değiştirilemez
END;
/
```

### OUT Parametresi

```sql
CREATE OR REPLACE PROCEDURE get_emp_details(
    p_emp_id IN NUMBER,
    p_full_name OUT VARCHAR2,
    p_salary OUT NUMBER,
    p_dept_name OUT VARCHAR2
) IS
BEGIN
    SELECT e.first_name || ' ' || e.last_name,
           e.salary,
           d.department_name
    INTO p_full_name, p_salary, p_dept_name
    FROM employees e
    JOIN departments d ON e.department_id = d.department_id
    WHERE e.employee_id = p_emp_id;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        p_full_name := 'Bulunamadı';
        p_salary := 0;
        p_dept_name := 'Bilinmiyor';
END;
/
```

### IN OUT Parametresi

```sql
CREATE OR REPLACE PROCEDURE format_phone(p_phone IN OUT VARCHAR2) IS
BEGIN
    -- Sadece rakamları al
    p_phone := REGEXP_REPLACE(p_phone, '[^0-9]', '');

    -- Format uygula (5XX XXX XX XX)
    IF LENGTH(p_phone) = 11 AND SUBSTR(p_phone, 1, 1) = '0' THEN
        p_phone := SUBSTR(p_phone, 2); -- İlk 0'ı kaldır
    END IF;

    IF LENGTH(p_phone) = 10 THEN
        p_phone := SUBSTR(p_phone, 1, 3) || ' ' ||
                   SUBSTR(p_phone, 4, 3) || ' ' ||
                   SUBSTR(p_phone, 7, 2) || ' ' ||
                   SUBSTR(p_phone, 9, 2);
    END IF;
END;
/

-- Kullanım
DECLARE
    v_phone VARCHAR2(20) := '0(555) 123-45-67';
BEGIN
    DBMS_OUTPUT.PUT_LINE('Önce: ' || v_phone);
    format_phone(v_phone);
    DBMS_OUTPUT.PUT_LINE('Sonra: ' || v_phone);
END;
/
```

## 5. Procedure vs Function Karşılaştırması

| Özellik            | Procedure                        | Function                     |
| ------------------ | -------------------------------- | ---------------------------- |
| Değer Döndürme     | Hayır (OUT parametrelerle çıktı) | Evet (RETURN ile)            |
| SELECT'te Kullanım | Hayır                            | Evet                         |
| DML İşlemleri      | Evet                             | Sınırlı (güvenlik nedeniyle) |
| CALL Komutu        | Evet                             | Evet                         |
| SQL İfadelerinde   | Hayır                            | Evet                         |

## 6. Gerçek Dünya Örnekleri

### Audit Log Procedure

```sql
CREATE OR REPLACE PROCEDURE log_user_action(
    p_user_id IN NUMBER,
    p_action IN VARCHAR2,
    p_table_name IN VARCHAR2,
    p_record_id IN NUMBER
) IS
BEGIN
    INSERT INTO audit_log (
        log_date, user_id, action_type,
        table_name, record_id, session_id
    ) VALUES (
        SYSDATE, p_user_id, p_action,
        p_table_name, p_record_id, SYS_CONTEXT('USERENV', 'SESSIONID')
    );

    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        -- Log hatası uygulamayı durdurmasın
        NULL;
END;
/
```

### Email Validation Function

```sql
CREATE OR REPLACE FUNCTION is_valid_email(p_email VARCHAR2)
RETURN BOOLEAN IS
BEGIN
    RETURN REGEXP_LIKE(p_email, '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');
EXCEPTION
    WHEN OTHERS THEN
        RETURN FALSE;
END;
/

-- Kullanım
DECLARE
    v_email VARCHAR2(100) := 'test@example.com';
BEGIN
    IF is_valid_email(v_email) THEN
        DBMS_OUTPUT.PUT_LINE('Geçerli email');
    ELSE
        DBMS_OUTPUT.PUT_LINE('Geçersiz email');
    END IF;
END;
/
```

## 7. Performans İpuçları

### 1. Bulk Operations

```sql
CREATE OR REPLACE PROCEDURE bulk_salary_update(p_dept_id NUMBER) IS
    TYPE emp_id_array IS TABLE OF NUMBER;
    TYPE salary_array IS TABLE OF NUMBER;

    v_emp_ids emp_id_array;
    v_new_salaries salary_array;
BEGIN
    SELECT employee_id, salary * 1.1
    BULK COLLECT INTO v_emp_ids, v_new_salaries
    FROM employees
    WHERE department_id = p_dept_id;

    FORALL i IN 1..v_emp_ids.COUNT
        UPDATE employees
        SET salary = v_new_salaries(i)
        WHERE employee_id = v_emp_ids(i);

    COMMIT;
END;
/
```

### 2. Function Caching

```sql
CREATE OR REPLACE FUNCTION get_tax_rate(p_salary NUMBER)
RETURN NUMBER
DETERMINISTIC -- Aynı input için aynı output garanti edilir
IS
BEGIN
    RETURN CASE
        WHEN p_salary > 50000 THEN 0.25
        WHEN p_salary > 30000 THEN 0.20
        WHEN p_salary > 15000 THEN 0.15
        ELSE 0.10
    END;
END;
/
```

### 3. Oracle Function Modifiers

```sql
-- DETERMINISTIC: Aynı input için her zaman aynı output
CREATE OR REPLACE FUNCTION calculate_bonus(p_salary NUMBER, p_rate NUMBER)
RETURN NUMBER
DETERMINISTIC
IS
BEGIN
    RETURN p_salary * p_rate / 100;
END;
/

-- RESULT_CACHE: Sonuçları otomatik cache'ler (11g+)
CREATE OR REPLACE FUNCTION get_company_info(p_company_id NUMBER)
RETURN VARCHAR2
RESULT_CACHE
IS
    v_company_name VARCHAR2(100);
BEGIN
    SELECT company_name INTO v_company_name
    FROM companies
    WHERE company_id = p_company_id;

    RETURN v_company_name;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN 'Şirket Bulunamadı';
END;
/

-- PARALLEL_ENABLE: Paralel sorgulamayı destekler
CREATE OR REPLACE FUNCTION simple_calculation(p_number NUMBER)
RETURN NUMBER
PARALLEL_ENABLE
DETERMINISTIC
IS
BEGIN
    RETURN p_number * 2;
END;
/

-- AUTHID: Çalıştırma yetkisi belirleme
CREATE OR REPLACE FUNCTION get_user_data(p_user_id NUMBER)
RETURN VARCHAR2
AUTHID CURRENT_USER  -- Çağıran kullanıcının yetkileriyle çalışır
IS
    v_data VARCHAR2(1000);
BEGIN
    SELECT user_info INTO v_data
    FROM user_details
    WHERE user_id = p_user_id;

    RETURN v_data;
END;
/

-- PIPELINED: Table function için (büyük veri setleri)
CREATE TYPE number_table AS TABLE OF NUMBER;
/

CREATE OR REPLACE FUNCTION get_numbers(p_start NUMBER, p_end NUMBER)
RETURN number_table
PIPELINED
IS
BEGIN
    FOR i IN p_start..p_end LOOP
        PIPE ROW(i);
    END LOOP;
    RETURN;
END;
/

-- Kullanım
SELECT * FROM TABLE(get_numbers(1, 10));
```

### 4. Advanced Parameter Features

```sql
-- DEFAULT değerlerle procedure
CREATE OR REPLACE PROCEDURE generate_report(
    p_start_date IN DATE DEFAULT SYSDATE - 30,
    p_end_date IN DATE DEFAULT SYSDATE,
    p_format IN VARCHAR2 DEFAULT 'PDF',
    p_email_list IN VARCHAR2 DEFAULT NULL
) IS
BEGIN
    DBMS_OUTPUT.PUT_LINE('Rapor oluşturuluyor...');
    DBMS_OUTPUT.PUT_LINE('Başlangıç: ' || TO_CHAR(p_start_date, 'DD-MON-YYYY'));
    DBMS_OUTPUT.PUT_LINE('Bitiş: ' || TO_CHAR(p_end_date, 'DD-MON-YYYY'));
    DBMS_OUTPUT.PUT_LINE('Format: ' || p_format);

    IF p_email_list IS NOT NULL THEN
        DBMS_OUTPUT.PUT_LINE('Email listesi: ' || p_email_list);
    END IF;
END;
/

-- Çağırma yöntemleri
BEGIN
    generate_report;  -- Tüm default değerler
    generate_report(SYSDATE - 7);  -- Sadece start_date
    generate_report(p_format => 'EXCEL');  -- Named parameter
    generate_report(p_start_date => SYSDATE - 15, p_format => 'CSV');
END;
/

-- NOCOPY hint: Büyük collections için performans
CREATE OR REPLACE PROCEDURE process_large_data(
    p_data IN OUT NOCOPY employee_mgmt.emp_array
) IS
BEGIN
    -- NOCOPY: Parametre değerini kopyalamaz, referans geçer
    -- Büyük collections için çok daha hızlı
    FOR i IN 1..p_data.COUNT LOOP
        p_data(i).salary := p_data(i).salary * 1.1;
    END LOOP;
END;
/
```

## Best Practices

1. **Naming Convention**: `get_`, `set_`, `calculate_`, `validate_` prefixleri kullan
2. **Error Handling**: Her procedure/function'da exception handling yap
3. **Documentation**: Başta açıklama yorum satırları ekle
4. **Parameter Validation**: Giriş parametrelerini kontrol et
5. **Transaction Management**: COMMIT/ROLLBACK işlemlerini doğru yönet

## Pratik Kullanım Alanları

1. **Data Processing**: Toplu veri işleme operasyonları
2. **Business Logic**: İş kurallarının merkezi yönetimi
3. **Integration**: Farklı sistemler arası veri alışverişi
4. **Reporting**: Karmaşık rapor hesaplamaları
5. **Maintenance**: Sistem bakım rutinleri

**Sonraki Bölümde:** Package yapısını ve cursor kullanımını öğreneceğiz.
