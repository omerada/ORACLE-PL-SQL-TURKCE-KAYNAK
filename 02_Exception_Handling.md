# Exception Handling ve Hata Yönetimi

## 1. Exception Nedir?

Exception (İstisna), program çalışırken oluşan hatalardır. PL/SQL'de bu hatalar yakalanıp kontrollü bir şekilde yönetilebilir.

**Neden Önemli?**

- Uygulamanın çökmesini önler
- Kullanıcıya anlamlı hata mesajları verir
- Log kayıtları tutar
- Sistemin kararlılığını sağlar

## 2. Exception Türleri

### Sistem Tanımlı (Predefined) Exceptions

Oracle'ın önceden tanımladığı yaygın hatalar:

```sql
DECLARE
    v_maas employees.salary%TYPE;
BEGIN
    SELECT salary INTO v_maas
    FROM employees
    WHERE employee_id = 999; -- Böyle bir ID yok

    DBMS_OUTPUT.PUT_LINE('Maaş: ' || v_maas);
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Çalışan bulunamadı!');
    WHEN TOO_MANY_ROWS THEN
        DBMS_OUTPUT.PUT_LINE('Birden fazla kayıt bulundu!');
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Beklenmeyen hata: ' || SQLERRM);
END;
/
```

### Yaygın Predefined Exceptions

- **NO_DATA_FOUND**: SELECT sorgusu sonuç döndürmedi
- **TOO_MANY_ROWS**: SELECT INTO birden fazla satır döndürdü
- **ZERO_DIVIDE**: Sıfıra bölme hatası
- **VALUE_ERROR**: Veri tipi uyumsuzluğu
- **DUP_VAL_ON_INDEX**: Unique constraint ihlali

## 3. Kullanıcı Tanımlı Exceptions

```sql
DECLARE
    v_maas NUMBER;
    maas_dusuk_exception EXCEPTION; -- Kendi exception'ımız
BEGIN
    SELECT salary INTO v_maas
    FROM employees
    WHERE employee_id = 100;

    IF v_maas < 5000 THEN
        RAISE maas_dusuk_exception; -- Exception'ı tetikliyoruz
    END IF;

    DBMS_OUTPUT.PUT_LINE('Maaş uygun: ' || v_maas);
EXCEPTION
    WHEN maas_dusuk_exception THEN
        DBMS_OUTPUT.PUT_LINE('Hata: Maaş çok düşük! (' || v_maas || ')');
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Çalışan bulunamadı!');
END;
/
```

## 4. RAISE_APPLICATION_ERROR

Özel hata kodları ve mesajları oluşturmak için:

```sql
DECLARE
    v_departman_id NUMBER := 999;
    v_sayac NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_sayac
    FROM departments
    WHERE department_id = v_departman_id;

    IF v_sayac = 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Departman ID ' || v_departman_id || ' bulunamadı!');
    END IF;

    DBMS_OUTPUT.PUT_LINE('Departman mevcut');
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Hata Kodu: ' || SQLCODE);
        DBMS_OUTPUT.PUT_LINE('Hata Mesajı: ' || SQLERRM);
END;
/
```

**RAISE_APPLICATION_ERROR Kuralları:**

- Hata kodu -20000 ile -20999 arasında olmalı
- Mesaj 2048 karakteri geçemez

## 5. PRAGMA EXCEPTION_INIT

Oracle hata numaralarını kendi exception'larımızla eşleştirmek için:

```sql
DECLARE
    foreign_key_error EXCEPTION;
    PRAGMA EXCEPTION_INIT(foreign_key_error, -2291);

    v_emp_id NUMBER := 9999;
    v_dept_id NUMBER := 999;
BEGIN
    INSERT INTO employees (employee_id, department_id, first_name, last_name, email, hire_date, job_id)
    VALUES (v_emp_id, v_dept_id, 'Test', 'User', 'test@email.com', SYSDATE, 'IT_PROG');

EXCEPTION
    WHEN foreign_key_error THEN
        DBMS_OUTPUT.PUT_LINE('Hata: Geçersiz departman ID: ' || v_dept_id);
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Diğer hata: ' || SQLERRM);
END;
/
```

## 6. Exception Propagation

Exception'lar iç bloklardan dış bloklara yayılır:

```sql
DECLARE
    v_sonuc NUMBER;
BEGIN
    BEGIN -- İç blok
        v_sonuc := 10 / 0; -- Hata oluşacak
    EXCEPTION
        WHEN ZERO_DIVIDE THEN
            DBMS_OUTPUT.PUT_LINE('İç blokta yakalandı');
            RAISE; -- Exception'ı dış bloğa yayıyor
    END;

    DBMS_OUTPUT.PUT_LINE('Bu satır çalışmayacak');
EXCEPTION
    WHEN ZERO_DIVIDE THEN
        DBMS_OUTPUT.PUT_LINE('Dış blokta yakalandı');
END;
/
```

## 7. Pratik Exception Handling Örneği

Gerçek iş senaryosunda exception handling:

```sql
CREATE OR REPLACE PROCEDURE employee_salary_update(
    p_emp_id IN NUMBER,
    p_new_salary IN NUMBER
) IS
    v_current_salary NUMBER;
    v_min_salary NUMBER;
    v_max_salary NUMBER;

    salary_too_low EXCEPTION;
    salary_too_high EXCEPTION;
    employee_not_found EXCEPTION;
BEGIN
    -- Mevcut maaş bilgilerini al
    SELECT salary INTO v_current_salary
    FROM employees
    WHERE employee_id = p_emp_id;

    -- İş pozisyonu için min-max maaş kontrolü
    SELECT min_salary, max_salary
    INTO v_min_salary, v_max_salary
    FROM jobs j, employees e
    WHERE j.job_id = e.job_id
    AND e.employee_id = p_emp_id;

    -- Maaş kontrolleri
    IF p_new_salary < v_min_salary THEN
        RAISE salary_too_low;
    ELSIF p_new_salary > v_max_salary THEN
        RAISE salary_too_high;
    END IF;

    -- Maaş güncelleme
    UPDATE employees
    SET salary = p_new_salary,
        last_update_date = SYSDATE
    WHERE employee_id = p_emp_id;

    COMMIT;
    DBMS_OUTPUT.PUT_LINE('Maaş başarıyla güncellendi: ' || p_new_salary);

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE_APPLICATION_ERROR(-20001, 'Çalışan ID ' || p_emp_id || ' bulunamadı');
    WHEN salary_too_low THEN
        RAISE_APPLICATION_ERROR(-20002, 'Yeni maaş minimum tutardan düşük: ' || v_min_salary);
    WHEN salary_too_high THEN
        RAISE_APPLICATION_ERROR(-20003, 'Yeni maaş maximum tutardan yüksek: ' || v_max_salary);
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE_APPLICATION_ERROR(-20999, 'Beklenmeyen hata: ' || SQLERRM);
END;
/
```

## 8. Exception Handling Best Practices

### 1. Spesifik Exception'ları Önce Yaz

```sql
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        -- Spesifik işlem
    WHEN VALUE_ERROR THEN
        -- Spesifik işlem
    WHEN OTHERS THEN
        -- Genel işlem
```

### 2. Meaningful Error Messages

```sql
WHEN NO_DATA_FOUND THEN
    RAISE_APPLICATION_ERROR(-20001,
        'Çalışan bulunamadı. ID: ' || p_employee_id ||
        ', Tarih: ' || TO_CHAR(SYSDATE, 'DD/MM/YYYY HH24:MI:SS'));
```

### 3. Logging

```sql
WHEN OTHERS THEN
    INSERT INTO error_log (error_date, error_code, error_message, procedure_name)
    VALUES (SYSDATE, SQLCODE, SQLERRM, 'EMPLOYEE_SALARY_UPDATE');
    COMMIT;
    RAISE;
```

## Pratik Kullanım Alanları

1. **Data Validation**: Veri girişi kontrolleri
2. **Business Rules**: İş kuralı ihlallerini yakalama
3. **Integration**: Dış sistemlerle entegrasyon hatalarını yönetme
4. **Maintenance**: Sistem bakım işlemlerinde hata yönetimi
5. **Reporting**: Rapor oluşturma sırasında veri eksikliklerini yönetme

**Sonraki Bölümde:** Procedure ve Function'ları öğreneceğiz.
