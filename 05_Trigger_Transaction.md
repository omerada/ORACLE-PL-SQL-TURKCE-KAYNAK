# Trigger ve Transaction Yönetimi

## 1. Trigger Nedir?

Trigger, veritabanında belirli olaylar gerçekleştiğinde otomatik olarak çalışan özel PL/SQL blokları です。

**Ne Zaman Kullanılır?**

- Veri bütünlüğü kontrolü
- Audit (denetim) kayıtları
- Otomatik hesaplamalar
- İş kuralları uygulaması
- Güvenlik kontrolleri

**Trigger Türleri:**

- **BEFORE**: İşlem öncesi
- **AFTER**: İşlem sonrası
- **INSTEAD OF**: View'lar için

## 2. DML Trigger'lar

### BEFORE INSERT Trigger

```sql
CREATE OR REPLACE TRIGGER employees_before_insert
    BEFORE INSERT ON employees
    FOR EACH ROW
BEGIN
    -- ID otomatik atama
    IF :NEW.employee_id IS NULL THEN
        SELECT employees_seq.NEXTVAL INTO :NEW.employee_id FROM DUAL;
    END IF;

    -- Email büyük harfe çevir
    :NEW.email := UPPER(:NEW.email);

    -- Hire date kontrolü
    IF :NEW.hire_date IS NULL THEN
        :NEW.hire_date := SYSDATE;
    END IF;

    -- Maaş kontrolü
    IF :NEW.salary < 1000 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Maaş minimum 1000 olmalı');
    END IF;
END;
/
```

### AFTER INSERT Trigger (Audit)

```sql
CREATE OR REPLACE TRIGGER employees_audit_insert
    AFTER INSERT ON employees
    FOR EACH ROW
BEGIN
    INSERT INTO employee_audit (
        action_type,
        employee_id,
        action_date,
        action_by,
        old_salary,
        new_salary
    ) VALUES (
        'INSERT',
        :NEW.employee_id,
        SYSDATE,
        USER,
        NULL,
        :NEW.salary
    );
END;
/
```

### BEFORE UPDATE Trigger

```sql
CREATE OR REPLACE TRIGGER employees_before_update
    BEFORE UPDATE ON employees
    FOR EACH ROW
BEGIN
    -- Last update bilgisini güncelle
    :NEW.last_update_date := SYSDATE;
    :NEW.last_update_by := USER;

    -- Maaş değişimi kontrolü
    IF :NEW.salary != :OLD.salary THEN
        -- %50'den fazla artış kontrolü
        IF :NEW.salary > :OLD.salary * 1.5 THEN
            RAISE_APPLICATION_ERROR(-20002,
                'Maaş artışı %50 den fazla olamaz. Eski: ' || :OLD.salary ||
                ', Yeni: ' || :NEW.salary);
        END IF;

        -- Maaş düşüş kontrolü
        IF :NEW.salary < :OLD.salary * 0.9 THEN
            RAISE_APPLICATION_ERROR(-20003,
                'Maaş %10 dan fazla düşürülemez');
        END IF;
    END IF;

    -- Email değişimi logu
    IF :NEW.email != :OLD.email THEN
        INSERT INTO email_change_log (
            employee_id, old_email, new_email, change_date, changed_by
        ) VALUES (
            :NEW.employee_id, :OLD.email, :NEW.email, SYSDATE, USER
        );
    END IF;
END;
/
```

### BEFORE DELETE Trigger

```sql
CREATE OR REPLACE TRIGGER employees_before_delete
    BEFORE DELETE ON employees
    FOR EACH ROW
DECLARE
    v_order_count NUMBER;
BEGIN
    -- Aktif siparişi olan çalışan silinemez
    SELECT COUNT(*) INTO v_order_count
    FROM orders
    WHERE sales_rep_id = :OLD.employee_id
    AND order_status = 'ACTIVE';

    IF v_order_count > 0 THEN
        RAISE_APPLICATION_ERROR(-20004,
            'Aktif siparişi olan çalışan silinemez. Sipariş sayısı: ' || v_order_count);
    END IF;

    -- Silme işlemini logla
    INSERT INTO employee_audit (
        action_type, employee_id, action_date, action_by,
        old_salary, employee_name
    ) VALUES (
        'DELETE', :OLD.employee_id, SYSDATE, USER,
        :OLD.salary, :OLD.first_name || ' ' || :OLD.last_name
    );
END;
/
```

## 3. Compound Trigger

Birden fazla timing event'i tek trigger'da yönetmek için:

```sql
CREATE OR REPLACE TRIGGER employees_compound_audit
    FOR INSERT OR UPDATE OR DELETE ON employees
    COMPOUND TRIGGER

    -- Global değişkenler
    TYPE audit_array IS TABLE OF employee_audit%ROWTYPE;
    g_audit_records audit_array := audit_array();

    -- BEFORE STATEMENT
    BEFORE STATEMENT IS
    BEGIN
        DBMS_OUTPUT.PUT_LINE('Employees tablosunda toplu işlem başlıyor: ' ||
                           TO_CHAR(SYSDATE, 'HH24:MI:SS'));
        g_audit_records.DELETE; -- Array'i temizle
    END BEFORE STATEMENT;

    -- BEFORE EACH ROW
    BEFORE EACH ROW IS
    BEGIN
        IF INSERTING THEN
            :NEW.created_date := SYSDATE;
            :NEW.created_by := USER;
        ELSIF UPDATING THEN
            :NEW.last_update_date := SYSDATE;
            :NEW.last_update_by := USER;
        END IF;
    END BEFORE EACH ROW;

    -- AFTER EACH ROW
    AFTER EACH ROW IS
        v_audit_rec employee_audit%ROWTYPE;
    BEGIN
        -- Audit kaydı hazırla
        v_audit_rec.employee_id := COALESCE(:NEW.employee_id, :OLD.employee_id);
        v_audit_rec.action_date := SYSDATE;
        v_audit_rec.action_by := USER;

        IF INSERTING THEN
            v_audit_rec.action_type := 'INSERT';
            v_audit_rec.new_salary := :NEW.salary;
        ELSIF UPDATING THEN
            v_audit_rec.action_type := 'UPDATE';
            v_audit_rec.old_salary := :OLD.salary;
            v_audit_rec.new_salary := :NEW.salary;
        ELSIF DELETING THEN
            v_audit_rec.action_type := 'DELETE';
            v_audit_rec.old_salary := :OLD.salary;
            v_audit_rec.employee_name := :OLD.first_name || ' ' || :OLD.last_name;
        END IF;

        -- Array'e ekle
        g_audit_records.EXTEND;
        g_audit_records(g_audit_records.COUNT) := v_audit_rec;
    END AFTER EACH ROW;

    -- AFTER STATEMENT
    AFTER STATEMENT IS
    BEGIN
        -- Toplu audit insert
        FORALL i IN 1..g_audit_records.COUNT
            INSERT INTO employee_audit VALUES g_audit_records(i);

        DBMS_OUTPUT.PUT_LINE('Audit kayıtları kaydedildi: ' || g_audit_records.COUNT || ' adet');
    END AFTER STATEMENT;

END employees_compound_audit;
/
```

## 4. Instead Of Trigger (View'lar için)

```sql
-- Karmaşık view oluşturalım
CREATE OR REPLACE VIEW employee_department_view AS
SELECT e.employee_id,
       e.first_name || ' ' || e.last_name as full_name,
       e.email,
       e.salary,
       d.department_name,
       d.department_id
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

-- Instead of trigger
CREATE OR REPLACE TRIGGER employee_dept_view_iou
    INSTEAD OF UPDATE ON employee_department_view
    FOR EACH ROW
BEGIN
    -- Employees tablosunu güncelle
    UPDATE employees
    SET salary = :NEW.salary,
        email = :NEW.email
    WHERE employee_id = :NEW.employee_id;

    -- Eğer departman değiştiyse
    IF :NEW.department_id != :OLD.department_id THEN
        UPDATE employees
        SET department_id = :NEW.department_id
        WHERE employee_id = :NEW.employee_id;

        -- Departman değişikliği logu
        INSERT INTO dept_change_log (
            employee_id, old_dept_id, new_dept_id, change_date
        ) VALUES (
            :NEW.employee_id, :OLD.department_id, :NEW.department_id, SYSDATE
        );
    END IF;
END;
/
```

## 5. Transaction Yönetimi

### COMMIT ve ROLLBACK

```sql
DECLARE
    v_error_count NUMBER := 0;
BEGIN
    -- Savepoint oluştur
    SAVEPOINT before_salary_update;

    FOR emp_rec IN (SELECT employee_id, salary FROM employees WHERE department_id = 50) LOOP
        BEGIN
            UPDATE employees
            SET salary = salary * 1.1
            WHERE employee_id = emp_rec.employee_id;

        EXCEPTION
            WHEN OTHERS THEN
                v_error_count := v_error_count + 1;
                DBMS_OUTPUT.PUT_LINE('Hata (ID: ' || emp_rec.employee_id || '): ' || SQLERRM);
        END;
    END LOOP;

    IF v_error_count > 0 THEN
        ROLLBACK TO before_salary_update;
        DBMS_OUTPUT.PUT_LINE('İşlem geri alındı. Hata sayısı: ' || v_error_count);
    ELSE
        COMMIT;
        DBMS_OUTPUT.PUT_LINE('Tüm maaşlar başarıyla güncellendi');
    END IF;
END;
/
```

### Autonomous Transaction

```sql
CREATE OR REPLACE PROCEDURE log_error(
    p_error_code VARCHAR2,
    p_error_message VARCHAR2,
    p_procedure_name VARCHAR2
) IS
    PRAGMA AUTONOMOUS_TRANSACTION; -- Bağımsız transaction
BEGIN
    INSERT INTO error_log (
        log_date, error_code, error_message, procedure_name, user_name
    ) VALUES (
        SYSDATE, p_error_code, p_error_message, p_procedure_name, USER
    );

    COMMIT; -- Bu commit ana transaction'ı etkilemez
END;
/

-- Kullanım
CREATE OR REPLACE PROCEDURE risky_operation IS
BEGIN
    -- Riskli işlem
    UPDATE employees SET salary = salary * 2 WHERE department_id = 999;

    IF SQL%ROWCOUNT = 0 THEN
        log_error('NO_DEPT', 'Departman bulunamadı', 'RISKY_OPERATION');
        RAISE_APPLICATION_ERROR(-20001, 'İşlem başarısız');
    END IF;

    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        log_error(SQLCODE, SQLERRM, 'RISKY_OPERATION');
        ROLLBACK;
        RAISE;
END;
/
```

## 6. Locking Stratejileri

### Pessimistic Locking

```sql
DECLARE
    CURSOR emp_cursor IS
        SELECT employee_id, salary
        FROM employees
        WHERE department_id = 50
        FOR UPDATE NOWAIT; -- Hemen kilitle, beklememe

    v_new_salary NUMBER;
BEGIN
    FOR emp_rec IN emp_cursor LOOP
        -- Kompleks hesaplama
        v_new_salary := emp_rec.salary * 1.1;

        UPDATE employees
        SET salary = v_new_salary
        WHERE CURRENT OF emp_cursor;
    END LOOP;

    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        IF SQLCODE = -54 THEN -- Resource busy
            DBMS_OUTPUT.PUT_LINE('Kayıtlar başka kullanıcı tarafından kilitlenmiş');
        END IF;
        ROLLBACK;
        RAISE;
END;
/
```

### Optimistic Locking

```sql
CREATE TABLE employees_opt (
    employee_id NUMBER PRIMARY KEY,
    first_name VARCHAR2(50),
    salary NUMBER,
    version_number NUMBER DEFAULT 1,
    last_update_date DATE DEFAULT SYSDATE
);

-- Optimistic update procedure
CREATE OR REPLACE PROCEDURE update_employee_optimistic(
    p_emp_id NUMBER,
    p_new_salary NUMBER,
    p_version NUMBER
) IS
    v_current_version NUMBER;
BEGIN
    -- Mevcut version kontrolü
    SELECT version_number INTO v_current_version
    FROM employees_opt
    WHERE employee_id = p_emp_id;

    IF v_current_version != p_version THEN
        RAISE_APPLICATION_ERROR(-20005,
            'Kayıt başka kullanıcı tarafından değiştirilmiş. ' ||
            'Beklenen version: ' || p_version ||
            ', Mevcut version: ' || v_current_version);
    END IF;

    UPDATE employees_opt
    SET salary = p_new_salary,
        version_number = version_number + 1,
        last_update_date = SYSDATE
    WHERE employee_id = p_emp_id
    AND version_number = p_version;

    IF SQL%ROWCOUNT = 0 THEN
        RAISE_APPLICATION_ERROR(-20006, 'Kayıt güncellenemedi');
    END IF;

    COMMIT;
END;
/
```

## 7. Gerçek Dünya Trigger Örneği

### Comprehensive Audit System

```sql
-- Audit tablosu
CREATE TABLE table_audit (
    audit_id NUMBER GENERATED ALWAYS AS IDENTITY,
    table_name VARCHAR2(100),
    operation_type VARCHAR2(10),
    pk_value VARCHAR2(100),
    column_name VARCHAR2(100),
    old_value CLOB,
    new_value CLOB,
    change_date DATE DEFAULT SYSDATE,
    changed_by VARCHAR2(100) DEFAULT USER,
    session_id VARCHAR2(100),
    ip_address VARCHAR2(50)
);

-- Generic audit trigger
CREATE OR REPLACE TRIGGER products_audit_trigger
    AFTER INSERT OR UPDATE OR DELETE ON products
    FOR EACH ROW
DECLARE
    v_operation VARCHAR2(10);
    v_pk_value VARCHAR2(100);

    PROCEDURE audit_column(p_column_name VARCHAR2, p_old_value VARCHAR2, p_new_value VARCHAR2) IS
    BEGIN
        IF p_old_value != p_new_value OR (p_old_value IS NULL AND p_new_value IS NOT NULL)
           OR (p_old_value IS NOT NULL AND p_new_value IS NULL) THEN
            INSERT INTO table_audit (
                table_name, operation_type, pk_value, column_name,
                old_value, new_value, session_id, ip_address
            ) VALUES (
                'PRODUCTS', v_operation, v_pk_value, p_column_name,
                p_old_value, p_new_value,
                SYS_CONTEXT('USERENV', 'SESSIONID'),
                SYS_CONTEXT('USERENV', 'IP_ADDRESS')
            );
        END IF;
    END audit_column;
BEGIN
    IF INSERTING THEN
        v_operation := 'INSERT';
        v_pk_value := :NEW.product_id;

        audit_column('PRODUCT_NAME', NULL, :NEW.product_name);
        audit_column('UNIT_PRICE', NULL, TO_CHAR(:NEW.unit_price));
        audit_column('CATEGORY_ID', NULL, TO_CHAR(:NEW.category_id));

    ELSIF UPDATING THEN
        v_operation := 'UPDATE';
        v_pk_value := :NEW.product_id;

        audit_column('PRODUCT_NAME', :OLD.product_name, :NEW.product_name);
        audit_column('UNIT_PRICE', TO_CHAR(:OLD.unit_price), TO_CHAR(:NEW.unit_price));
        audit_column('CATEGORY_ID', TO_CHAR(:OLD.category_id), TO_CHAR(:NEW.category_id));

    ELSIF DELETING THEN
        v_operation := 'DELETE';
        v_pk_value := :OLD.product_id;

        audit_column('PRODUCT_NAME', :OLD.product_name, NULL);
        audit_column('UNIT_PRICE', TO_CHAR(:OLD.unit_price), NULL);
        audit_column('CATEGORY_ID', TO_CHAR(:OLD.category_id), NULL);
    END IF;
END;
/
```

## 8. Trigger Performance Tips

### 1. Avoid Recursive Triggers

```sql
-- Yanlış yaklaşım (sonsuz döngü riski)
CREATE OR REPLACE TRIGGER bad_trigger
    AFTER UPDATE ON employees
    FOR EACH ROW
BEGIN
    UPDATE employees SET last_update = SYSDATE WHERE employee_id = :NEW.employee_id;
END;
/

-- Doğru yaklaşım
CREATE OR REPLACE TRIGGER good_trigger
    BEFORE UPDATE ON employees
    FOR EACH ROW
BEGIN
    :NEW.last_update := SYSDATE; -- Aynı row üzerinde değişiklik
END;
/
```

### 2. Bulk Operations

```sql
CREATE OR REPLACE TRIGGER bulk_audit_trigger
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW
BEGIN
    -- Tek tek insert yerine queue'ya ekle
    audit_pkg.add_to_queue(
        table_name => 'ORDERS',
        operation => CASE
            WHEN INSERTING THEN 'INSERT'
            WHEN UPDATING THEN 'UPDATE'
            WHEN DELETING THEN 'DELETE'
        END,
        pk_value => COALESCE(:NEW.order_id, :OLD.order_id)
    );
END;
/
```

## Best Practices

1. **Minimal Logic**: Trigger'larda minimum işlem yapın
2. **Error Handling**: Her trigger'da exception handling
3. **Performance**: Bulk operations tercih edin
4. **Recursive Protection**: Sonsuz döngü koruması
5. **Logging**: Trigger aktivitelerini loglayin
6. **Testing**: Trigger'ları kapsamlı test edin

## Pratik Kullanım Alanları

1. **Data Auditing**: Veri değişiklik takibi
2. **Business Rules**: Otomatik iş kuralı uygulama
3. **Data Validation**: Karmaşık veri kontrolleri
4. **Cache Management**: Cache invalidation
5. **Integration**: Sistem entegrasyonu trigger'ları
6. **Security**: Güvenlik kontrolleri ve logging

**Sonraki Bölümde:** Oracle Forms mimarisi ve çalışma yapısını öğreneceğiz.
