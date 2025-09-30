# PL/SQL Temelleri

## 1. PL/SQL Nedir?

PL/SQL (Procedural Language/Structured Query Language), Oracle'ın SQL'e prosedürel programlama yetenekleri ekleyen dilidir. Normal SQL sadece veri sorgulamaya odaklanırken, PL/SQL ile:

- **Değişken tanımlama**
- **Koşullu işlemler**
- **Döngüler**
- **Hata yönetimi**
- **Fonksiyon ve prosedürler**

yazabilirsiniz.

**Neden Önemli?**

- Veritabanı seviyesinde iş mantığı yazılabilir
- Performans artışı (network trafiği azalır)
- Güvenlik (SQL injection riskini azaltır)
- Merkezi yönetim (iş kuralları DB'de)

## 2. Blok Yapısı

PL/SQL'de her kod bloğu 3 bölümden oluşur:

```sql
DECLARE
    -- Değişken tanımlamaları (isteğe bağlı)
BEGIN
    -- Çalıştırılacak kodlar (zorunlu)
EXCEPTION
    -- Hata yönetimi (isteğe bağlı)
END;
/
```

**Basit Örnek:**

```sql
DECLARE
    v_mesaj VARCHAR2(50) := 'Merhaba PL/SQL';
BEGIN
    -- DBMS_OUTPUT.PUT_LINE: Console'a metin yazdırmak için kullanılır
    -- Tıpkı Java'daki System.out.println() gibi
    -- Sadece SQL*Plus, SQL Developer gibi araçlarda görünür
    DBMS_OUTPUT.PUT_LINE(v_mesaj);

    -- DBMS_OUTPUT.PUT: Aynı satıra yazar (yeni satır eklemez)
    DBMS_OUTPUT.PUT('Satır 1 ');
    DBMS_OUTPUT.PUT('devam ediyor... ');
    DBMS_OUTPUT.NEW_LINE; -- Yeni satıra geç

    -- DBMS_OUTPUT.ENABLE: Output'u aktif eder (varsayılan: aktif)
    DBMS_OUTPUT.ENABLE(1000000);  -- Buffer boyutu (max 1MB)

    -- DBMS_OUTPUT.DISABLE: Output'u kapatır
    -- DBMS_OUTPUT.DISABLE;
END;
/

-- SQL*Plus/SQL Developer'da output'u görmek için:
-- SET SERVEROUTPUT ON
```

## 3. Değişkenler ve Veri Tipleri

### Temel Veri Tipleri

```sql
DECLARE
    -- Veri tipleri ve başlangıç değerleri
    v_sayi NUMBER(10,2) := 150.75;      -- Sayı: 10 basamak, 2 ondalık
    v_metin VARCHAR2(100) := 'Oracle';   -- Değişken uzunluk string (max 100)
    v_tarih DATE := SYSDATE;             -- Tarih ve saat
    v_boolean BOOLEAN := TRUE;           -- TRUE, FALSE, NULL değerler alabilir
    v_char CHAR(5) := 'ABCDE';          -- Sabit uzunluk (5 karakter, eksikse boşluk ekler)

    -- NULL değerler
    v_null_sayi NUMBER;                  -- Başlangıçta NULL
    v_not_null VARCHAR2(50) NOT NULL := 'Boş olamaz'; -- NULL olmasına izin vermez

    -- Constant tanımlama (değiştirilemez)
    c_pi CONSTANT NUMBER := 3.14159;    -- Değişmez değer
    c_company CONSTANT VARCHAR2(20) := 'ORACLE';
BEGIN
    DBMS_OUTPUT.PUT_LINE('Sayı: ' || v_sayi);
    DBMS_OUTPUT.PUT_LINE('Metin: ' || v_metin);
    DBMS_OUTPUT.PUT_LINE('Tarih: ' || TO_CHAR(v_tarih, 'DD/MM/YYYY HH24:MI:SS'));

    -- Boolean değerler direkt yazdırılamaz, çevirmek gerekir
    IF v_boolean THEN
        DBMS_OUTPUT.PUT_LINE('Boolean: TRUE');
    ELSE
        DBMS_OUTPUT.PUT_LINE('Boolean: FALSE');
    END IF;

    -- NULL kontrolü
    IF v_null_sayi IS NULL THEN
        DBMS_OUTPUT.PUT_LINE('v_null_sayi NULL değerde');
    END IF;

    DBMS_OUTPUT.PUT_LINE('Pi değeri: ' || c_pi);
END;
/
```

### %TYPE Kullanımı

Veritabanı sütunuyla aynı tipte değişken tanımlamak için:

```sql
DECLARE
    -- %TYPE: Tablonun sütun tipini kullanır
    -- Eğer employees.first_name VARCHAR2(50) ise, v_emp_name de VARCHAR2(50) olur
    v_emp_name employees.first_name%TYPE;
    v_emp_salary employees.salary%TYPE;
    v_dept_id employees.department_id%TYPE;

    -- %ROWTYPE: Tablonun tüm sütunlarını içeren record oluşturur
    v_emp_record employees%ROWTYPE;
BEGIN
    -- Tek sütun alma
    SELECT first_name, salary, department_id
    INTO v_emp_name, v_emp_salary, v_dept_id
    FROM employees
    WHERE employee_id = 100;

    DBMS_OUTPUT.PUT_LINE(v_emp_name || ' maaşı: ' || v_emp_salary);
    DBMS_OUTPUT.PUT_LINE('Departman ID: ' || v_dept_id);

    -- Tüm satırı alma
    SELECT * INTO v_emp_record
    FROM employees
    WHERE employee_id = 101;

    -- Record alanına erişim (nokta notasyonu)
    DBMS_OUTPUT.PUT_LINE('Kayıt: ' || v_emp_record.first_name || ' ' ||
                        v_emp_record.last_name || ', Maaş: ' || v_emp_record.salary);
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Hiçbir kayıt bulunamadı!');
    WHEN TOO_MANY_ROWS THEN
        DBMS_OUTPUT.PUT_LINE('Birden fazla kayıt bulundu!');
END;
/
```

**Neden %TYPE Kullanmalı?**

- Tablo yapısı değişirse kod otomatik uyum sağlar
- Tip uyumsuzluğu hatalarını önler

## 4. Koşullu İfadeler

### IF-THEN-ELSE

```sql
DECLARE
    v_maas NUMBER := 5000;
    v_kategori VARCHAR2(20);
BEGIN
    IF v_maas > 10000 THEN
        v_kategori := 'Yüksek';
    ELSIF v_maas > 5000 THEN
        v_kategori := 'Orta';
    ELSE
        v_kategori := 'Düşük';
    END IF;

    DBMS_OUTPUT.PUT_LINE('Maaş kategorisi: ' || v_kategori);
END;
/
```

### CASE İfadesi

```sql
DECLARE
    v_gun NUMBER := 3;
    v_gun_adi VARCHAR2(20);
BEGIN
    v_gun_adi := CASE v_gun
        WHEN 1 THEN 'Pazartesi'
        WHEN 2 THEN 'Salı'
        WHEN 3 THEN 'Çarşamba'
        WHEN 4 THEN 'Perşembe'
        WHEN 5 THEN 'Cuma'
        ELSE 'Hafta sonu'
    END;

    DBMS_OUTPUT.PUT_LINE('Bugün: ' || v_gun_adi);
END;
/
```

## 5. Döngüler

### FOR Döngüsü

```sql
-- Basit sayısal döngü
BEGIN
    -- FOR i IN 1..5: i değişkeni 1'den 5'e kadar artar
    -- i değişkenini ayrıca tanımlamaya gerek yok
    FOR i IN 1..5 LOOP
        DBMS_OUTPUT.PUT_LINE('Sayı: ' || i);
    END LOOP;
END;
/

-- Ters döngü (azalan)
BEGIN
    -- REVERSE ile 5'ten 1'e kadar azalır
    FOR i IN REVERSE 1..5 LOOP
        DBMS_OUTPUT.PUT_LINE('Azalan: ' || i);
    END LOOP;
END;
/

-- Cursor FOR döngüsü (veritabanı kayıtları için)
BEGIN
    -- Her çalışan için döngü
    FOR emp_rec IN (SELECT employee_id, first_name, salary FROM employees) LOOP
        DBMS_OUTPUT.PUT_LINE('ID: ' || emp_rec.employee_id ||
                           ', İsim: ' || emp_rec.first_name ||
                           ', Maaş: ' || emp_rec.salary);
    END LOOP;
END;
/

-- Dinamik sınırlar ile
DECLARE
    v_start NUMBER := 10;
    v_end NUMBER := 15;
BEGIN
    FOR i IN v_start..v_end LOOP
        DBMS_OUTPUT.PUT_LINE('Dinamik: ' || i);
    END LOOP;
END;
/
```

### WHILE Döngüsü

```sql
DECLARE
    v_sayac NUMBER := 1;
BEGIN
    -- WHILE: Koşul TRUE olduğu sürece devam eder
    -- Koşul başta kontrol edilir, FALSE ise hiç çalışmayabilir
    WHILE v_sayac <= 5 LOOP
        DBMS_OUTPUT.PUT_LINE('Sayaç: ' || v_sayac);

        -- Sayıcıyı arttırmayı unutma! (yoksa sonsuz döngü)
        v_sayac := v_sayac + 1;
    END LOOP;

    DBMS_OUTPUT.PUT_LINE('Döngü bitti, son değer: ' || v_sayac);
END;
/

-- Daha karmaşık örnek
DECLARE
    v_toplam NUMBER := 0;
    v_sayi NUMBER := 1;
BEGIN
    -- 1+2+3+...+n toplamı 100'ü geçene kadar
    WHILE v_toplam <= 100 LOOP
        v_toplam := v_toplam + v_sayi;
        DBMS_OUTPUT.PUT_LINE(v_sayi || ' eklendi, toplam: ' || v_toplam);
        v_sayi := v_sayi + 1;
    END LOOP;

    DBMS_OUTPUT.PUT_LINE('Son toplam: ' || v_toplam);
END;
/
```

### LOOP-EXIT Döngüsü

```sql
DECLARE
    v_toplam NUMBER := 0;
    v_i NUMBER := 1;
BEGIN
    -- LOOP: Sonsuz döngü, çıkış koşulu döngü içinde
    -- En az bir kez mutlaka çalışır (do-while gibi)
    LOOP
        v_toplam := v_toplam + v_i;
        DBMS_OUTPUT.PUT_LINE('Eklenen: ' || v_i || ', Toplam: ' || v_toplam);
        v_i := v_i + 1;

        -- EXIT WHEN: Koşul TRUE olunca döngüden çık
        EXIT WHEN v_toplam > 100;

        -- Alternatif: IF koşulu ile EXIT
        -- IF v_toplam > 100 THEN
        --     EXIT;
        -- END IF;
    END LOOP;

    DBMS_OUTPUT.PUT_LINE('Son toplam: ' || v_toplam);
END;
/

-- İç içe döngü ve CONTINUE örneği
DECLARE
    v_i NUMBER;
    v_j NUMBER;
BEGIN
    v_i := 1;
    LOOP
        v_j := 1;
        DBMS_OUTPUT.PUT_LINE('Dış döngü: ' || v_i);

        LOOP
            -- CONTINUE WHEN: Koşul TRUE ise bu iteration'u atla (11g+)
            CONTINUE WHEN MOD(v_j, 2) = 0;  -- Çift sayıları atla

            DBMS_OUTPUT.PUT_LINE('  İç döngü: ' || v_j);
            v_j := v_j + 1;

            EXIT WHEN v_j > 5;  -- İç döngüden çık
        END LOOP;

        v_i := v_i + 1;
        EXIT WHEN v_i > 3;      -- Dış döngüden çık
    END LOOP;
END;
/
```

## Pratik Kullanım Alanları

1. **Veri Doğrulama**: Kayıt eklemeden önce iş kurallarını kontrol etme

   ```sql
   -- Email formatı kontrolü örneği
   IF v_email NOT LIKE '%@%.%' THEN
       RAISE_APPLICATION_ERROR(-20001, 'Geçersiz email formatı');
   END IF;
   ```

2. **Toplu İşlemler**: Binlerce kaydı işleme

   ```sql
   -- Tüm çalışanlara zam verme
   FOR emp IN (SELECT employee_id FROM employees) LOOP
       UPDATE employees SET salary = salary * 1.1
       WHERE employee_id = emp.employee_id;
   END LOOP;
   ```

3. **Raporlama**: Karmaşık hesaplamalar yapma

   ```sql
   -- Departman bazında maaş analizi
   FOR dept IN (SELECT department_id FROM departments) LOOP
       SELECT AVG(salary), COUNT(*) INTO v_avg, v_count
       FROM employees WHERE department_id = dept.department_id;
       DBMS_OUTPUT.PUT_LINE('Dept: ' || dept.department_id ||
                           ' Avg: ' || v_avg || ' Count: ' || v_count);
   END LOOP;
   ```

4. **Veri Migrasyon**: Eski sistemden yeni sisteme veri aktarma
5. **Trigger'lar**: Tablo değişikliklerinde otomatik işlemler

## Temel SQL vs PL/SQL Komutları

### Cursor'lar (İleri Konularda Detaylanacak)

```sql
-- Basit cursor kullanımı
DECLARE
    CURSOR emp_cursor IS
        SELECT employee_id, first_name, salary FROM employees;
    v_emp_id NUMBER;
    v_name VARCHAR2(50);
    v_salary NUMBER;
BEGIN
    OPEN emp_cursor;
    LOOP
        FETCH emp_cursor INTO v_emp_id, v_name, v_salary;
        EXIT WHEN emp_cursor%NOTFOUND;

        DBMS_OUTPUT.PUT_LINE(v_emp_id || ': ' || v_name || ' - ' || v_salary);
    END LOOP;
    CLOSE emp_cursor;
END;
/
```

### Temel Veritabanı İşlemleri

```sql
DECLARE
    v_emp_id NUMBER := 1001;
    v_name VARCHAR2(50) := 'Ahmet Yılmaz';
    v_salary NUMBER := 5000;
BEGIN
    -- INSERT işlemi
    INSERT INTO employees (employee_id, first_name, salary, hire_date)
    VALUES (v_emp_id, v_name, v_salary, SYSDATE);

    -- UPDATE işlemi
    UPDATE employees
    SET salary = salary * 1.1
    WHERE employee_id = v_emp_id;

    -- Kaç satır etkilendiğini kontrol et
    DBMS_OUTPUT.PUT_LINE(SQL%ROWCOUNT || ' satır güncellendi');

    -- DELETE işlemi
    DELETE FROM employees WHERE employee_id = v_emp_id;

    -- Transaction kontrolü
    COMMIT;  -- Değişiklikleri kaydet
    -- ROLLBACK;  -- Değişiklikleri geri al

EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;  -- Hata durumunda geri al
        DBMS_OUTPUT.PUT_LINE('Hata: ' || SQLERRM);
END;
/
```

## Önemli Notlar ve Best Practices

### Syntax Kuralları

- **PL/SQL blokları `/` ile biter** (SQL\*Plus ve SQL Developer için)
- **Her satır `;` ile biter** (statements için)
- **Büyük/küçük harf duyarsız** ama okunabilirlik için standart kullanın
- **Yorum satırları**: `--` (tek satır) veya `/* */` (çok satır)

### Naming Conventions (İsimlendirme Kuralları)

- **Variables**: `v_variable_name` (v\_ prefix)
- **Constants**: `c_constant_name` (c\_ prefix)
- **Parameters**: `p_parameter_name` (p\_ prefix)
- **Cursors**: `cursor_name` (suffix: `_cur` veya `_cursor`)

```sql
DECLARE
    v_employee_name VARCHAR2(100);    -- Variable
    c_tax_rate CONSTANT NUMBER := 0.18;  -- Constant
    p_dept_id NUMBER;                 -- Parameter (procedure'da)
BEGIN
    NULL;  -- Hiçbir şey yapma (placeholder)
END;
/
```

### Debug ve Test

```sql
-- SET SERVEROUTPUT ON;  -- SQL*Plus'ta output görmek için
-- DBMS_OUTPUT.PUT_LINE('Debug: v_value = ' || v_value);
```

**Sonraki Bölümde:** Exception Handling ve hata yönetimi öğreneceğiz.
