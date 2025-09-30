# Oracle Forms Mimarisi ve Temel Kavramlar

## 1. Oracle Forms Nedir?

Oracle Forms, Oracle Corporation tarafından geliştirilen, client-server mimarisinde çalışan masaüstü uygulama geliştirme platformudur.

**Temel Özellikler:**

- RAD (Rapid Application Development) ortamı
- PL/SQL entegrasyonu
- Oracle veritabanı ile doğrudan bağlantı
- Rich client interface
- Master-detail ilişkileri
- Built-in validation ve navigation

**Ne Zaman Kullanılır?**

- Veri girişi yoğun uygulamalar
- Karmaşık iş mantığı gerektiren formlar
- Legacy sistemler
- Desktop tabanlı enterprise uygulamalar

## 2. Oracle Forms Mimarisi

### 3-Tier Mimari

```
Client Tier (Browser/Java)
    ↕
Middle Tier (Application Server)
    ↕
Database Tier (Oracle Database)
```

### Bileşenler

1. **Forms Builder**: Geliştirme ortamı (.fmb dosyaları)
2. **Forms Compiler**: .fmb'yi .fmx'e dönüştürür
3. **Forms Runtime**: Uygulamayı çalıştırır
4. **Application Server**: Web deployment için

## 3. Form Yapısı Hiyerarşisi

```
FORM (Form)
├── MODULE (Module)
├── WINDOW (Pencere)
├── CANVAS (Tuval)
└── BLOCK (Blok)
    ├── ITEM (Alan/Field)
    └── TRIGGER (Tetikleyici)
```

### Form Hiyerarşisi Örneği

```
EMPLOYEE_FORM
├── MAIN_WINDOW
├── DETAIL_WINDOW
├── MAIN_CANVAS
├── DETAIL_CANVAS
├── EMPLOYEE_BLOCK
│   ├── EMP_ID (Item)
│   ├── FIRST_NAME (Item)
│   ├── LAST_NAME (Item)
│   ├── SALARY (Item)
│   └── WHEN-NEW-RECORD-INSTANCE (Trigger)
└── DEPARTMENT_BLOCK
    ├── DEPT_ID (Item)
    ├── DEPT_NAME (Item)
    └── POST-QUERY (Trigger)
```

## 4. Temel Kavramlar

### Form (.fmb/.fmx)

Kullanıcı arayüzü ve iş mantığını içeren ana yapı.

### Window (Pencere)

Formun görsel container'ı. Bir form birden fazla window içerebilir.

```sql
-- Window özellikleri
WINDOW_NAME: MAIN_WINDOW
Title: Employee Management
Width: 800
Height: 600
Modal: False
```

### Canvas (Tuval)

UI elementlerinin yerleştirildiği yüzey. Window içinde gösterilir.

**Canvas Türleri:**

- **Content Canvas**: Ana içerik alanı
- **Stacked Canvas**: Diğer canvas'ların üzerine konur
- **Tab Canvas**: Sekme yapısı
- **Toolbar Canvas**: Araç çubuğu

### Block (Blok)

Veritabanı tablosu veya view ile ilişkili veri grubunu temsil eder.

**Block Türleri:**

- **Data Block**: Veritabanı tablosuna bağlı
- **Control Block**: Sadece kontrol amaçlı

### Item (Alan)

Kullanıcının veri girdiği veya gördüğü form elemanları.

**Item Türleri:**

- **Text Item**: Metin girişi
- **Display Item**: Sadece görüntüleme
- **List Item**: Dropdown liste
- **Radio Group**: Radio button grubu
- **Checkbox**: Onay kutusu
- **Button**: Buton
- **Image**: Resim

## 5. Forms Veri Mimarisi

### Master-Detail İlişkisi

```sql
-- Master Block: DEPARTMENTS
SELECT dept_id, dept_name, location
FROM departments

-- Detail Block: EMPLOYEES
SELECT emp_id, first_name, last_name, salary, dept_id
FROM employees
WHERE dept_id = :DEPARTMENTS.DEPT_ID
```

### Block Coordination

Master block'ta seçilen kayda göre detail block otomatik filtrelenir.

```sql
-- Department seçildiğinde
-- DEPARTMENTS block -> DEPT_ID = 10
-- EMPLOYEES block otomatik query: WHERE dept_id = 10
```

## 6. Form Lifecycle (Yaşam Döngüsü)

### Form Startup Sequence

```
1. PRE-FORM trigger
2. Form initialization
3. WHEN-NEW-FORM-INSTANCE trigger
4. First block navigation
5. WHEN-NEW-BLOCK-INSTANCE trigger
6. First record creation
7. WHEN-NEW-RECORD-INSTANCE trigger
8. First item navigation
9. WHEN-NEW-ITEM-INSTANCE trigger
```

### Navigation Hierarchy

```
Form → Block → Record → Item
```

## 7. PL/SQL'in Forms'daki Rolü

### Built-in Subprograms

Forms'un hazır PL/SQL fonksiyonları:

```sql
-- Navigation
GO_BLOCK('EMPLOYEE');
GO_ITEM('EMPLOYEE.FIRST_NAME');
NEXT_RECORD;
PREVIOUS_RECORD;

-- DML Operations
COMMIT_FORM;
ROLLBACK_FORM;
EXECUTE_QUERY;
CREATE_RECORD;
DELETE_RECORD;

-- Message Display
MESSAGE('Kayıt başarıyla kaydedildi');
BELL; -- Ses çıkarır
```

### Form Level Variables

```sql
-- Global variables
:GLOBAL.USER_ID := 'JOHN_DOE';
:GLOBAL.CURRENT_DATE := TO_CHAR(SYSDATE, 'DD/MM/YYYY');

-- Parameter variables
:PARAMETER.DEPT_ID := 10;

-- System variables
IF :SYSTEM.FORM_STATUS = 'CHANGED' THEN
    COMMIT_FORM;
END IF;
```

## 8. Trigger Konsepti

### Trigger Kategorileri

**1. Navigation Triggers**

```sql
-- WHEN-NEW-FORM-INSTANCE
BEGIN
    :GLOBAL.USER_NAME := GET_APPLICATION_PROPERTY(USERNAME);
    GO_BLOCK('EMPLOYEE');
END;

-- WHEN-NEW-BLOCK-INSTANCE
BEGIN
    IF :SYSTEM.CURSOR_BLOCK = 'EMPLOYEE' THEN
        EXECUTE_QUERY;
    END IF;
END;

-- WHEN-NEW-RECORD-INSTANCE
BEGIN
    IF :SYSTEM.MODE = 'ENTER' THEN
        :EMPLOYEE.HIRE_DATE := SYSDATE;
        :EMPLOYEE.CREATED_BY := :GLOBAL.USER_NAME;
    END IF;
END;
```

**2. Validation Triggers**

```sql
-- WHEN-VALIDATE-ITEM (Salary field)
BEGIN
    IF :EMPLOYEE.SALARY < 1000 THEN
        MESSAGE('Maaş 1000 den küçük olamaz');
        RAISE FORM_TRIGGER_FAILURE;
    END IF;

    IF :EMPLOYEE.SALARY > 50000 THEN
        IF SHOW_ALERT('HIGH_SALARY_ALERT') = ALERT_BUTTON1 THEN
            NULL; -- Onaylandı
        ELSE
            RAISE FORM_TRIGGER_FAILURE;
        END IF;
    END IF;
END;

-- POST-CHANGE
BEGIN
    -- Maaş değiştiğinde bonus hesapla
    :EMPLOYEE.BONUS := :EMPLOYEE.SALARY * 0.1;
END;
```

**3. Transaction Triggers**

```sql
-- PRE-INSERT
BEGIN
    SELECT employee_seq.NEXTVAL INTO :EMPLOYEE.EMP_ID FROM DUAL;
    :EMPLOYEE.CREATED_DATE := SYSDATE;
    :EMPLOYEE.CREATED_BY := :GLOBAL.USER_NAME;
END;

-- POST-INSERT
BEGIN
    -- Audit log
    INSERT INTO emp_audit (emp_id, action, action_date, action_by)
    VALUES (:EMPLOYEE.EMP_ID, 'INSERT', SYSDATE, :GLOBAL.USER_NAME);
END;

-- PRE-UPDATE
BEGIN
    :EMPLOYEE.LAST_UPDATE_DATE := SYSDATE;
    :EMPLOYEE.LAST_UPDATE_BY := :GLOBAL.USER_NAME;
END;

-- PRE-DELETE
DECLARE
    v_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_count
    FROM orders
    WHERE sales_rep_id = :EMPLOYEE.EMP_ID;

    IF v_count > 0 THEN
        MESSAGE('Bu çalışanın siparişleri var. Silinemez.');
        RAISE FORM_TRIGGER_FAILURE;
    END IF;
END;
```

## 9. Query ve DML İşlemleri

### Dynamic Query

```sql
-- WHEN-BUTTON-PRESSED (Search Button)
DECLARE
    v_where_clause VARCHAR2(1000) := 'WHERE 1=1 ';
BEGIN
    -- Dynamic WHERE clause oluştur
    IF :CONTROL.SEARCH_NAME IS NOT NULL THEN
        v_where_clause := v_where_clause ||
            'AND UPPER(first_name || '' '' || last_name) LIKE UPPER(''%' ||
            :CONTROL.SEARCH_NAME || '%'') ';
    END IF;

    IF :CONTROL.SEARCH_DEPT IS NOT NULL THEN
        v_where_clause := v_where_clause ||
            'AND department_id = ' || :CONTROL.SEARCH_DEPT || ' ';
    END IF;

    IF :CONTROL.SEARCH_MIN_SALARY IS NOT NULL THEN
        v_where_clause := v_where_clause ||
            'AND salary >= ' || :CONTROL.SEARCH_MIN_SALARY || ' ';
    END IF;

    -- Block'un WHERE clause'unu set et
    SET_BLOCK_PROPERTY('EMPLOYEE', DEFAULT_WHERE, v_where_clause);

    -- Query'yi çalıştır
    GO_BLOCK('EMPLOYEE');
    EXECUTE_QUERY;
END;
```

### Batch Processing

```sql
-- WHEN-BUTTON-PRESSED (Mass Update Button)
DECLARE
    v_current_record NUMBER := 1;
    v_total_records NUMBER;
BEGIN
    v_total_records := GET_BLOCK_PROPERTY('EMPLOYEE', QUERY_HITS);

    FIRST_RECORD;
    LOOP
        -- Her kayıt için maaş artışı
        :EMPLOYEE.SALARY := :EMPLOYEE.SALARY * 1.1;
        :EMPLOYEE.LAST_UPDATE_DATE := SYSDATE;

        EXIT WHEN v_current_record = v_total_records;

        NEXT_RECORD;
        v_current_record := v_current_record + 1;
    END LOOP;

    COMMIT_FORM;
    MESSAGE(v_total_records || ' kayıt güncellendi');
END;
```

## 10. Forms Servisleri

### Database Interaction

```sql
-- Direct SQL execution
DECLARE
    v_bonus NUMBER;
BEGIN
    SELECT AVG(salary) * 0.1 INTO v_bonus
    FROM employees
    WHERE department_id = :EMPLOYEE.DEPARTMENT_ID;

    :EMPLOYEE.CALCULATED_BONUS := v_bonus;
END;

-- Stored procedure call
BEGIN
    calculate_employee_bonus(
        p_emp_id => :EMPLOYEE.EMP_ID,
        p_bonus => :EMPLOYEE.BONUS
    );
END;
```

### File Operations

```sql
-- Report çağırma
DECLARE
    v_rep_status VARCHAR2(20);
BEGIN
    v_rep_status := RUN_PRODUCT(
        REPORTS,
        'emp_report',
        SYNCHRONOUS,
        RUNTIME,
        FILESYSTEM,
        pl_id,
        'P_DEPT_ID=' || :EMPLOYEE.DEPARTMENT_ID
    );

    IF v_rep_status != 'FINISHED' THEN
        MESSAGE('Rapor oluşturulamadı');
    END IF;
END;
```

## 11. Error Handling

### Form Level Exception Handling

```sql
-- ON-ERROR trigger
DECLARE
    v_error_type VARCHAR2(10) := ERROR_TYPE;
    v_error_code NUMBER := ERROR_CODE;
    v_error_text VARCHAR2(200) := ERROR_TEXT;
BEGIN
    IF v_error_code = 40001 THEN -- Unique constraint
        MESSAGE('Bu kayıt zaten mevcut');
        BELL;
    ELSIF v_error_code = 40202 THEN -- Record changed by another user
        MESSAGE('Kayıt başka kullanıcı tarafından değiştirilmiş');
        EXECUTE_QUERY;
    ELSE
        MESSAGE('Hata: ' || v_error_text);
    END IF;
END;

-- ON-MESSAGE trigger
BEGIN
    IF MESSAGE_TYPE = 'ERROR' THEN
        BELL;
    END IF;

    -- Default message handling
    NULL;
END;
```

## 12. Best Practices

### Performance

1. **Query Optimization**: WHERE clause'ları optimize edin
2. **Block Coordination**: Gereksiz query'leri önleyin
3. **Array Processing**: Bulk operations kullanın

### User Experience

1. **Navigation**: Mantıklı tab order
2. **Validation**: Immediate feedback
3. **Error Messages**: Anlaşılır mesajlar

### Maintenance

1. **Code Organization**: Trigger'ları düzenli tutun
2. **Documentation**: Kod açıklamaları
3. **Version Control**: Source control kullanın

## Pratik Kullanım Alanları

1. **Data Entry Forms**: Veri girişi ekranları
2. **Master-Detail Forms**: İlişkili veri yönetimi
3. **Lookup Forms**: Veri seçimi için popup'lar
4. **Report Parameters**: Rapor parametresi girişi
5. **Administrative Tools**: Sistem yönetim araçları

**Sonraki Bölümde:** Oracle Forms'tan Java Spring Microservices'e modernizasyon sürecini öğreneceğiz.
