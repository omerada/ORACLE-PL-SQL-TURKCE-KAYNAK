# Oracle APEX - Modern Web Development

## 1. Oracle APEX Nedir?

**Basit Tanım:** Oracle Application Express (APEX), Oracle Database üzerinde web uygulamaları geliştirmek için kullanılan low-code development platformudur.

**Low-Code Ne Demek:** Az kod yazarak, görsel araçlarla hızlı uygulama geliştirme yaklaşımı.

**Neden Önemli:** Oracle Forms'un modern, web tabanlı alternatifidir.

**Ana Özellikler:**

- **Low-Code Development**: Minimum kod ile uygulama geliştirme
  - **Anlamı:** Drag-drop ile form, tablo, grafik oluşturabilirsiniz
- **Web-Based**: Modern, responsive web uygulamaları
  - **Anlamı:** Herhangi bir browser'dan erişilebilir, mobil uyumlu
- **Integrated**: Oracle Database ile tam entegrasyon
  - **Anlamı:** PL/SQL kodlarınızı direkt kullanabilirsiniz
- **Rapid Development**: Hızlı prototipleme ve geliştirme
  - **Anlamı:** Günler içinde çalışan uygulama oluşturabilirsiniz
- **No Additional Licensing**: Oracle Database lisansıyla dahil
  - **Anlamı:** Ekstra maliyet yok, Oracle DB'niz varsa APEX de var

**Oracle Forms'tan Farkları:**

```
Oracle Forms                    Oracle APEX
═══════════════               ═══════════════
Client-Server                 Web-Based (İnternet üzerinden)
Desktop UI                    Responsive Web UI (Mobil uyumlu)
Java Runtime Required         Browser-Based (Sadece browser)
Limited Mobile Support        Mobile-First Design (Mobil öncelikli)
Complex Deployment            Simple Web Deployment (Kolay yayınlama)
Legacy Technology             Modern Technology (Çağdaş)
```

**Sonuç:** APEX, Forms'un modern dünyadaki karşılığıdır.

## 2. APEX Architecture

### APEX Mimarisi

```
Web Browser (HTML5/CSS3/JavaScript)
                ↕
    Web Server (HTTP/HTTPS)
                ↕
    APEX Engine (PL/SQL)
                ↕
    Oracle Database
```

### APEX Components

```
APEX Workspace
├── Applications
│   ├── Pages
│   │   ├── Regions
│   │   ├── Items
│   │   └── Processes
│   ├── Shared Components
│   │   ├── Navigation
│   │   ├── Lists
│   │   └── LOVs
│   └── Supporting Objects
│       ├── Database Objects
│       └── Static Files
```

## 3. APEX vs Forms Comparison

### Development Approach

**Oracle Forms Approach:**

```sql
-- Forms'ta trigger-based development
-- WHEN-BUTTON-PRESSED trigger
DECLARE
    v_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_count
    FROM employees
    WHERE department_id = :BLOCK.DEPT_ID;

    :BLOCK.EMP_COUNT := v_count;
    MESSAGE('Found ' || v_count || ' employees');
END;
```

**APEX Approach:**

```sql
-- APEX'te PL/SQL process
-- Page Process: Calculate Employee Count
DECLARE
    v_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_count
    FROM employees
    WHERE department_id = :P1_DEPT_ID;

    :P1_EMP_COUNT := v_count;

    apex_application.g_print_success_message := 'Found ' || v_count || ' employees';
END;
```

### UI Development

**Forms UI Definition:**

```
Form Module
├── Canvas (Visual Design)
├── Block (Data Binding)
├── Items (Form Fields)
└── Triggers (Event Handling)
```

**APEX UI Definition:**

```
APEX Page
├── Page Template (Layout)
├── Regions (Content Areas)
├── Items (Input/Display)
├── Buttons (Actions)
└── Dynamic Actions (Client-side Events)
```

## 4. APEX Application Structure

### Creating a Workspace

```sql
-- APEX workspace creation (via APEX Administration)
-- 1. Navigate to APEX Administration
-- 2. Create Workspace
-- 3. Assign schema
-- 4. Set workspace details

-- Or via SQL (as APEX_ADMIN)
BEGIN
    APEX_INSTANCE_ADMIN.ADD_WORKSPACE(
        p_workspace_id => 1000,
        p_workspace => 'COMPANY_WS',
        p_primary_schema => 'HR',
        p_additional_schemas => 'HR:SALES'
    );
END;
/
```

### Sample Application Structure

```sql
-- Create sample tables for APEX demo
CREATE TABLE departments (
    dept_id NUMBER PRIMARY KEY,
    dept_name VARCHAR2(100) NOT NULL,
    location VARCHAR2(100),
    manager_id NUMBER,
    budget NUMBER(12,2),
    created_date DATE DEFAULT SYSDATE
);

CREATE TABLE employees (
    emp_id NUMBER PRIMARY KEY,
    first_name VARCHAR2(50),
    last_name VARCHAR2(50),
    email VARCHAR2(100) UNIQUE,
    phone VARCHAR2(20),
    hire_date DATE,
    job_title VARCHAR2(100),
    salary NUMBER(8,2),
    department_id NUMBER REFERENCES departments(dept_id),
    manager_id NUMBER,
    created_date DATE DEFAULT SYSDATE
);

-- Insert sample data
INSERT INTO departments VALUES (10, 'Information Technology', 'Istanbul', NULL, 500000, SYSDATE);
INSERT INTO departments VALUES (20, 'Human Resources', 'Ankara', NULL, 200000, SYSDATE);
INSERT INTO departments VALUES (30, 'Sales', 'Izmir', NULL, 300000, SYSDATE);

-- APEX-specific sequences
CREATE SEQUENCE dept_seq START WITH 100;
CREATE SEQUENCE emp_seq START WITH 1000;
```

## 5. APEX Page Types ve Components

### Interactive Report

```sql
-- SQL Query for Interactive Report
SELECT
    emp_id,
    first_name || ' ' || last_name as full_name,
    email,
    phone,
    hire_date,
    job_title,
    TO_CHAR(salary, '999,999.00') as formatted_salary,
    d.dept_name,
    CASE
        WHEN salary > 10000 THEN 'High'
        WHEN salary > 5000 THEN 'Medium'
        ELSE 'Low'
    END as salary_category
FROM employees e
JOIN departments d ON e.department_id = d.dept_id
WHERE (:P1_DEPT_ID IS NULL OR e.department_id = :P1_DEPT_ID)
AND (:P1_SEARCH IS NULL OR
     UPPER(e.first_name || ' ' || e.last_name) LIKE '%' || UPPER(:P1_SEARCH) || '%')
ORDER BY e.last_name, e.first_name;
```

### Form Page

```sql
-- Form Page Process: Create Employee
-- Process Point: Processing, After Submit
DECLARE
    v_emp_id NUMBER;
BEGIN
    -- Generate new ID
    v_emp_id := emp_seq.NEXTVAL;

    -- Insert new employee
    INSERT INTO employees (
        emp_id, first_name, last_name, email, phone,
        hire_date, job_title, salary, department_id
    ) VALUES (
        v_emp_id, :P2_FIRST_NAME, :P2_LAST_NAME, :P2_EMAIL, :P2_PHONE,
        :P2_HIRE_DATE, :P2_JOB_TITLE, :P2_SALARY, :P2_DEPARTMENT_ID
    );

    -- Set success message
    apex_application.g_print_success_message := 'Employee created successfully with ID: ' || v_emp_id;

    -- Clear form
    :P2_EMP_ID := v_emp_id;

EXCEPTION
    WHEN DUP_VAL_ON_INDEX THEN
        apex_application.g_print_error_message := 'Email address already exists';
    WHEN OTHERS THEN
        apex_application.g_print_error_message := 'Error creating employee: ' || SQLERRM;
END;
```

### Charts ve Dashboards

```sql
-- Chart Query: Department Budget vs Actual Spending
SELECT
    d.dept_name as label,
    d.budget as "Budget",
    NVL(SUM(e.salary * 12), 0) as "Annual Salary Cost",
    d.budget - NVL(SUM(e.salary * 12), 0) as "Remaining Budget"
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.department_id
GROUP BY d.dept_id, d.dept_name, d.budget
ORDER BY d.dept_name;

-- Dashboard Metrics
-- Metric 1: Total Employees
SELECT COUNT(*) FROM employees;

-- Metric 2: Average Salary
SELECT ROUND(AVG(salary), 2) FROM employees;

-- Metric 3: Active Departments
SELECT COUNT(*) FROM departments WHERE dept_id IN (SELECT DISTINCT department_id FROM employees);
```

## 6. APEX ile PL/SQL Integration

### Application Processes

```sql
-- Application Process: Session State Management
-- Process Point: On New Session
BEGIN
    -- Set user preferences
    :G_USER_THEME := NVL(APEX_UTIL.GET_PREFERENCE('USER_THEME'), 'UNIVERSAL_THEME');
    :G_USER_LANG := NVL(APEX_UTIL.GET_PREFERENCE('USER_LANGUAGE'), 'en');

    -- Log session start
    INSERT INTO user_sessions (
        session_id, user_name, login_time, ip_address
    ) VALUES (
        :APP_SESSION, :APP_USER, SYSDATE,
        APEX_APPLICATION.G_X_FORWARDED_FOR
    );

    COMMIT;
END;
```

### Dynamic Actions

```sql
-- Dynamic Action: Cascade LOV (Department -> Employees)
-- Event: Change
-- Selection Type: Item
-- Item: P1_DEPARTMENT_ID

-- True Action: Refresh
-- Affected Elements: P1_EMPLOYEE_ID

-- Employee LOV Query
SELECT
    first_name || ' ' || last_name as display_value,
    emp_id as return_value
FROM employees
WHERE department_id = :P1_DEPARTMENT_ID
ORDER BY last_name, first_name;
```

### APEX Collections

```sql
-- Working with APEX Collections
-- Create collection
BEGIN
    apex_collection.create_or_truncate_collection(
        p_collection_name => 'EMPLOYEE_SELECTION'
    );

    -- Add members to collection
    FOR emp_rec IN (
        SELECT emp_id, first_name, last_name, salary
        FROM employees
        WHERE department_id = :P1_DEPT_ID
    ) LOOP
        apex_collection.add_member(
            p_collection_name => 'EMPLOYEE_SELECTION',
            p_c001 => emp_rec.emp_id,
            p_c002 => emp_rec.first_name,
            p_c003 => emp_rec.last_name,
            p_n001 => emp_rec.salary
        );
    END LOOP;
END;

-- Query collection
SELECT
    c001 as emp_id,
    c002 as first_name,
    c003 as last_name,
    n001 as salary,
    APEX_ITEM.CHECKBOX(1, c001) as selector
FROM apex_collections
WHERE collection_name = 'EMPLOYEE_SELECTION'
ORDER BY c003, c002;
```

## 7. Security in APEX

### Authentication

```sql
-- Custom Authentication Function
CREATE OR REPLACE FUNCTION custom_authenticate(
    p_username VARCHAR2,
    p_password VARCHAR2
) RETURN BOOLEAN IS
    v_count NUMBER;
    v_stored_hash VARCHAR2(100);
    v_input_hash VARCHAR2(100);
BEGIN
    -- Get stored password hash
    SELECT password_hash INTO v_stored_hash
    FROM app_users
    WHERE username = UPPER(p_username)
    AND status = 'ACTIVE';

    -- Hash input password
    v_input_hash := APEX_UTIL.GET_HASH(
        APEX_T_VARCHAR2(UPPER(p_username), p_password, 'APP_SALT')
    );

    -- Compare hashes
    IF v_stored_hash = v_input_hash THEN
        -- Update last login
        UPDATE app_users
        SET last_login = SYSDATE
        WHERE username = UPPER(p_username);
        COMMIT;

        RETURN TRUE;
    ELSE
        RETURN FALSE;
    END IF;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN FALSE;
    WHEN OTHERS THEN
        RETURN FALSE;
END;
/
```

### Authorization

```sql
-- Authorization Scheme: Department Manager
-- Scheme Type: PL/SQL Function Returning Boolean
DECLARE
    v_is_manager BOOLEAN := FALSE;
BEGIN
    SELECT CASE WHEN COUNT(*) > 0 THEN TRUE ELSE FALSE END
    INTO v_is_manager
    FROM departments
    WHERE manager_id = (
        SELECT emp_id FROM employees
        WHERE UPPER(first_name || '.' || last_name) = UPPER(:APP_USER)
    );

    RETURN v_is_manager;
END;

-- VPD Implementation for APEX
CREATE OR REPLACE FUNCTION apex_row_security(
    p_schema VARCHAR2,
    p_object VARCHAR2
) RETURN VARCHAR2 IS
    v_predicate VARCHAR2(4000);
BEGIN
    IF p_object = 'EMPLOYEES' THEN
        -- Users can only see employees in their department
        v_predicate := 'department_id IN (
            SELECT department_id FROM employees
            WHERE UPPER(first_name || ''.'' || last_name) = UPPER(''' || :APP_USER || ''')
        )';
    END IF;

    RETURN v_predicate;
END;
/
```

## 8. APEX vs Modern Web Development

### APEX Advantages

1. **Database Integration**: Native Oracle integration
2. **Rapid Development**: Visual development environment
3. **No Infrastructure**: Uses existing Oracle database
4. **Security**: Built-in security features
5. **Scalability**: Leverages Oracle's scalability

### APEX Limitations

1. **Oracle Dependency**: Requires Oracle Database
2. **Customization Limits**: UI customization constraints
3. **Learning Curve**: Specific to Oracle ecosystem
4. **Modern Frameworks**: Less flexible than React/Angular

### When to Choose APEX

**Choose APEX When:**

- Oracle Database centric applications
- Rapid prototyping needed
- Internal enterprise applications
- Limited frontend development resources
- Strong database business logic

**Choose Modern Stack When:**

- Multi-database support needed
- Complex UI/UX requirements
- Public-facing applications
- Microservices architecture
- Team has modern web skills

## 9. Migration Scenarios

### Forms to APEX Migration

```sql
-- Forms Block -> APEX Interactive Report
-- Forms Master-Detail -> APEX Master-Detail Pages

-- Sample migration: Employee Management Form

-- 1. Create APEX Application
-- 2. Create Interactive Report (List Page)
-- 3. Create Form Page (Edit Page)
-- 4. Migrate business logic

-- Forms trigger logic migration
-- Original Forms WHEN-VALIDATE-ITEM trigger:
/*
IF :EMPLOYEE.SALARY < 1000 THEN
    MESSAGE('Salary cannot be less than 1000');
    RAISE FORM_TRIGGER_FAILURE;
END IF;
*/

-- APEX equivalent: Item Validation
-- Validation Type: PL/SQL Expression
-- Expression: :P2_SALARY >= 1000
-- Error Message: Salary cannot be less than 1000
```

### APEX to Modern Web Migration

```sql
-- APEX business logic can be exposed as REST APIs
-- Enable RESTful Services on schema

-- REST Handler Example
-- URI Template: employees/{id}
-- HTTP Method: GET

DECLARE
    v_emp_json CLOB;
BEGIN
    SELECT JSON_OBJECT(
        'emp_id' VALUE emp_id,
        'full_name' VALUE first_name || ' ' || last_name,
        'email' VALUE email,
        'salary' VALUE salary,
        'department' VALUE (
            SELECT dept_name FROM departments
            WHERE dept_id = e.department_id
        )
    ) INTO v_emp_json
    FROM employees e
    WHERE emp_id = :id;

    HTP.P(v_emp_json);
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        HTP.P('{"error": "Employee not found"}');
        OWA_UTIL.STATUS_LINE(404);
END;
```

## 10. Best Practices

### APEX Development Best Practices

1. **Page Design**

   - Use consistent page templates
   - Implement responsive design
   - Optimize for mobile devices

2. **Performance**

   - Optimize SQL queries
   - Use APEX collections for complex operations
   - Implement proper caching

3. **Security**

   - Implement proper authentication
   - Use authorization schemes
   - Validate all inputs

4. **Maintenance**
   - Use version control (APEX export/import)
   - Document applications
   - Implement proper error handling

### Code Organization

```sql
-- Package for APEX utility functions
CREATE OR REPLACE PACKAGE apex_utils IS
    -- Navigation utilities
    PROCEDURE redirect_to_page(p_page_id NUMBER, p_item_values VARCHAR2 DEFAULT NULL);

    -- Message utilities
    PROCEDURE show_success(p_message VARCHAR2);
    PROCEDURE show_error(p_message VARCHAR2);

    -- Data utilities
    FUNCTION get_lov_display(p_return_value VARCHAR2, p_lov_query VARCHAR2) RETURN VARCHAR2;

    -- Validation utilities
    FUNCTION is_valid_email(p_email VARCHAR2) RETURN BOOLEAN;
    FUNCTION is_valid_phone(p_phone VARCHAR2) RETURN BOOLEAN;
END apex_utils;
/
```

## Modernization Roadmap

### Phase 1: Assessment

- [ ] Analyze existing Forms applications
- [ ] Identify migration candidates
- [ ] Evaluate APEX capabilities

### Phase 2: Pilot Project

- [ ] Select simple Forms application
- [ ] Migrate to APEX
- [ ] Gather user feedback

### Phase 3: Bulk Migration

- [ ] Develop migration standards
- [ ] Train development team
- [ ] Migrate applications systematically

### Phase 4: Modernization

- [ ] Evaluate modern web stack needs
- [ ] Implement REST APIs
- [ ] Consider cloud deployment

**Final Bölümde:** Güncellenmiş öğrenme yol haritasını sunacağız.
