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
    DBMS_OUTPUT.PUT_LINE(v_mesaj);
END;
/
```

## 3. Değişkenler ve Veri Tipleri

### Temel Veri Tipleri

```sql
DECLARE
    v_sayi NUMBER(10,2) := 150.75;
    v_metin VARCHAR2(100) := 'Oracle';
    v_tarih DATE := SYSDATE;
    v_boolean BOOLEAN := TRUE;
    v_char CHAR(5) := 'ABCDE';
BEGIN
    DBMS_OUTPUT.PUT_LINE('Sayı: ' || v_sayi);
    DBMS_OUTPUT.PUT_LINE('Metin: ' || v_metin);
    DBMS_OUTPUT.PUT_LINE('Tarih: ' || TO_CHAR(v_tarih, 'DD/MM/YYYY'));
END;
/
```

### %TYPE Kullanımı

Veritabanı sütunuyla aynı tipte değişken tanımlamak için:

```sql
DECLARE
    v_emp_name employees.first_name%TYPE;
    v_emp_salary employees.salary%TYPE;
BEGIN
    SELECT first_name, salary
    INTO v_emp_name, v_emp_salary
    FROM employees
    WHERE employee_id = 100;

    DBMS_OUTPUT.PUT_LINE(v_emp_name || ' maaşı: ' || v_emp_salary);
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
BEGIN
    FOR i IN 1..5 LOOP
        DBMS_OUTPUT.PUT_LINE('Sayı: ' || i);
    END LOOP;
END;
/
```

### WHILE Döngüsü

```sql
DECLARE
    v_sayac NUMBER := 1;
BEGIN
    WHILE v_sayac <= 5 LOOP
        DBMS_OUTPUT.PUT_LINE('Sayaç: ' || v_sayac);
        v_sayac := v_sayac + 1;
    END LOOP;
END;
/
```

### LOOP-EXIT Döngüsü

```sql
DECLARE
    v_toplam NUMBER := 0;
    v_i NUMBER := 1;
BEGIN
    LOOP
        v_toplam := v_toplam + v_i;
        v_i := v_i + 1;

        EXIT WHEN v_toplam > 100;
    END LOOP;

    DBMS_OUTPUT.PUT_LINE('Toplam: ' || v_toplam);
END;
/
```

## Pratik Kullanım Alanları

1. **Veri Doğrulama**: Kayıt eklemeden önce iş kurallarını kontrol etme
2. **Toplu İşlemler**: Binlerce kaydı işleme
3. **Raporlama**: Karmaşık hesaplamalar yapma
4. **Veri Migrasyon**: Eski sistemden yeni sisteme veri aktarma
5. **Trigger'lar**: Tablo değişikliklerinde otomatik işlemler

## Önemli Notlar

- PL/SQL blokları `/` ile biter
- `DBMS_OUTPUT.PUT_LINE` ile console'a yazı yazdırılır
- Değişken isimleri `v_` öneki ile başlatılması yaygın pratiktir
- Her satır `;` ile biter
- Yorum satırları `--` veya `/* */` ile yazılır

**Sonraki Bölümde:** Exception Handling ve hata yönetimi öğreneceğiz.
