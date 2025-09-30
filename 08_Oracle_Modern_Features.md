# Oracle Modern Features (19c/21c/23c)

## 1. Oracle Versiyonlarına Genel Bakış

### Oracle 19c (LTS - Long Term Support)

**Çıkış Tarihi:** 2019
**Desteklenen:** 2030'a kadar
**Ana Özellikler:**

- Automatic indexing
- Real-time SQL Plan Management
- SQL Macros
- Private Temporary Tables
- Blockchain Tables (21c'den backport)

### Oracle 21c (Innovation Release)

**Çıkış Tarihi:** 2021
**Ana Özellikler:**

- Native JSON Binary Format
- Blockchain Tables
- Immutable Tables
- JavaScript in Database
- Graph Analytics

### Oracle 23c (LTS)

**Çıkış Tarihi:** 2023
**Ana Özellikler:**

- SQL Domains
- Multi-dimensional Arrays
- Graph Property Graphs
- Vector Search
- Enhanced JSON support

## 2. JSON Support ve SQL/JSON Functions

### 19c JSON Enhancements

```sql
-- JSON_TABLE enhancements
CREATE TABLE products_json (
    id NUMBER,
    product_data JSON,
    CONSTRAINT check_json CHECK (product_data IS JSON)
);

-- JSON with constraints
ALTER TABLE products_json ADD CONSTRAINT product_name_required
    CHECK (JSON_EXISTS(product_data, '$.name'));

-- JSON indexing
CREATE INDEX idx_product_category
ON products_json (JSON_VALUE(product_data, '$.category'));

-- Complex JSON queries
SELECT p.id,
       JSON_VALUE(p.product_data, '$.name') as product_name,
       JSON_VALUE(p.product_data, '$.price' RETURNING NUMBER) as price,
       JSON_QUERY(p.product_data, '$.features[*]') as features
FROM products_json p
WHERE JSON_EXISTS(p.product_data, '$.category?(@=="Electronics")');
```

### 21c Native JSON Binary Format

```sql
-- JSON binary storage (faster processing)
CREATE TABLE orders_json (
    order_id NUMBER,
    order_data JSON
) STORAGE (JSON (order_data) STORE AS (FORMAT OSON));

-- JSON Schema validation
ALTER TABLE orders_json ADD CONSTRAINT order_schema_validation
CHECK (order_data IS JSON VALIDATE USING '{
    "type": "object",
    "properties": {
        "customer_id": {"type": "number"},
        "order_date": {"type": "string", "format": "date"},
        "items": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "product_id": {"type": "number"},
                    "quantity": {"type": "number", "minimum": 1}
                }
            }
        }
    },
    "required": ["customer_id", "order_date", "items"]
}');

-- JSON aggregation functions
SELECT
    JSON_ARRAYAGG(
        JSON_OBJECT(
            'product_name' VALUE product_name,
            'total_sales' VALUE SUM(quantity * price)
        )
    ) as sales_summary
FROM order_items
GROUP BY product_id;
```

### 23c Advanced JSON Features

```sql
-- JSON Relational Duality Views
CREATE JSON RELATIONAL DUALITY VIEW customer_orders_dv AS
    customers @insert @update @delete
    {
        customer_id: customer_id,
        name: customer_name,
        email: email,
        orders: orders @insert @update @delete
        [
            {
                order_id: order_id,
                order_date: order_date,
                total_amount: total_amount,
                items: order_items @insert @update @delete
                [
                    {
                        product_id: product_id,
                        quantity: quantity,
                        unit_price: unit_price
                    }
                ]
            }
        ]
    };

-- JSON Schema evolution
ALTER TABLE products_json MODIFY CONSTRAINT product_schema_validation
CHECK (product_data IS JSON VALIDATE USING '{
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "price": {"type": "number", "minimum": 0},
        "category": {"type": "string"},
        "tags": {"type": "array", "items": {"type": "string"}},
        "metadata": {"type": "object"}
    }
}');
```

## 3. Graph Database Capabilities

### Property Graph Support (21c+)

```sql
-- Create property graph
CREATE PROPERTY GRAPH company_graph
    VERTEX TABLES (
        employees KEY (employee_id)
            PROPERTIES (employee_id, first_name, last_name, email),
        departments KEY (department_id)
            PROPERTIES (department_id, department_name, location)
    )
    EDGE TABLES (
        employee_department
            SOURCE KEY (employee_id) REFERENCES employees
            DESTINATION KEY (department_id) REFERENCES departments
            PROPERTIES (start_date, role),
        employee_manager
            SOURCE KEY (employee_id) REFERENCES employees (employee_id)
            DESTINATION KEY (manager_id) REFERENCES employees (employee_id)
            PROPERTIES (reporting_since)
    );

-- Graph queries using PGQL
SELECT e1.first_name, e1.last_name, e2.first_name as manager_name
FROM GRAPH_TABLE(company_graph
    MATCH (e1:employees) -[r:employee_manager]-> (e2:employees)
    COLUMNS (e1.first_name, e1.last_name, e2.first_name)
);

-- Find all employees in same department
SELECT e1.first_name || ' ' || e1.last_name as employee,
       e2.first_name || ' ' || e2.last_name as colleague
FROM GRAPH_TABLE(company_graph
    MATCH (e1:employees) -[:employee_department]-> (d:departments) <-[:employee_department]- (e2:employees)
    WHERE e1.employee_id != e2.employee_id
    COLUMNS (e1.first_name, e1.last_name, e2.first_name, e2.last_name)
);
```

### Graph Analytics (23c)

```sql
-- PageRank algorithm
SELECT vertex_id, pagerank_value
FROM GRAPH_TABLE(company_graph
    MATCH (v)
    COLUMNS (VERTEX_ID(v), GRAPH_ALGORITHM('PAGERANK', v))
);

-- Shortest path analysis
SELECT path_length, path_vertices
FROM GRAPH_TABLE(company_graph
    MATCH TOP 5 SHORTEST (e1:employees) -[*]-> (e2:employees)
    WHERE e1.employee_id = 100 AND e2.employee_id = 200
    COLUMNS (PATH_LENGTH(), PATH_VERTICES())
);

-- Community detection
SELECT vertex_id, community_id
FROM GRAPH_TABLE(company_graph
    MATCH (v)
    COLUMNS (VERTEX_ID(v), GRAPH_ALGORITHM('LOUVAIN', v))
);
```

## 4. Machine Learning Integration

### Oracle Machine Learning (OML)

```sql
-- Create ML model for employee salary prediction
BEGIN
    DBMS_DATA_MINING.CREATE_MODEL(
        model_name => 'SALARY_PREDICTION_MODEL',
        mining_function => DBMS_DATA_MINING.REGRESSION,
        data_table_name => 'EMPLOYEE_ML_DATA',
        case_id_column_name => 'EMPLOYEE_ID',
        target_column_name => 'SALARY',
        settings_table_name => 'SALARY_MODEL_SETTINGS'
    );
END;
/

-- Model settings
CREATE TABLE salary_model_settings (
    setting_name VARCHAR2(30),
    setting_value VARCHAR2(4000)
);

INSERT INTO salary_model_settings VALUES ('ALGO_NAME', 'ALGO_GENERALIZED_LINEAR_MODEL');
INSERT INTO salary_model_settings VALUES ('PREP_AUTO', 'ON');
INSERT INTO salary_model_settings VALUES ('GLMS_SOLVER', 'DBMS_DATA_MINING.GLMS_SOLVER_QR');

-- Use model for predictions
SELECT employee_id,
       current_salary,
       PREDICTION(salary_prediction_model USING *) as predicted_salary,
       PREDICTION_PROBABILITY(salary_prediction_model USING *) as confidence
FROM employee_test_data;

-- Model evaluation
SELECT *
FROM TABLE(DBMS_DATA_MINING.GET_MODEL_DETAILS_GLM('SALARY_PREDICTION_MODEL'));
```

### AutoML Features (23c)

```sql
-- Automatic feature engineering
BEGIN
    DBMS_AUTO_ML.CREATE_MODEL(
        model_name => 'AUTO_CHURN_MODEL',
        table_name => 'CUSTOMER_DATA',
        target_column => 'CHURN_FLAG',
        model_type => 'CLASSIFICATION',
        auto_feature_generation => TRUE,
        auto_hyperparameter_tuning => TRUE
    );
END;
/

-- Model insights
SELECT feature_name, importance_score
FROM TABLE(DBMS_AUTO_ML.GET_FEATURE_IMPORTANCE('AUTO_CHURN_MODEL'))
ORDER BY importance_score DESC;

-- Automated model deployment
BEGIN
    DBMS_AUTO_ML.DEPLOY_MODEL(
        model_name => 'AUTO_CHURN_MODEL',
        deployment_name => 'CHURN_PREDICTION_SERVICE',
        enable_real_time_scoring => TRUE
    );
END;
/
```

## 5. Autonomous Database Features

### Automatic Indexing

```sql
-- Enable automatic indexing
ALTER DATABASE SET AUTO_INDEX = ON;

-- Monitor automatic index creation
SELECT index_name, table_name, auto_created, last_used
FROM USER_INDEXES
WHERE auto = 'YES';

-- Index usage statistics
SELECT sql_id, index_name, usage_count, last_used
FROM DBA_AUTO_INDEX_SQL_EXECUTIONS
WHERE username = USER;

-- Manual control over automatic indexes
-- Disable automatic indexing for specific table
ALTER TABLE employees SET AUTO_INDEX = OFF;

-- Accept or reject automatic index recommendations
BEGIN
    DBMS_AUTO_INDEX.CONFIGURE('AUTO_INDEX_MODE', 'IMPLEMENT');
    DBMS_AUTO_INDEX.CONFIGURE('AUTO_INDEX_REPORT_ONLY', 'FALSE');
END;
/
```

### SQL Plan Management Automation

```sql
-- Enable automatic SQL plan management
ALTER SYSTEM SET OPTIMIZER_CAPTURE_SQL_PLAN_BASELINES = TRUE;

-- View automatic plan baselines
SELECT sql_handle, plan_name, enabled, accepted, fixed
FROM DBA_SQL_PLAN_BASELINES
WHERE creator = 'AUTO_CAPTURE';

-- Automatic plan evolution
BEGIN
    DBMS_SPM.SET_EVOLVE_TASK_PARAMETER(
        task_name => 'SYS_AUTO_SPM_EVOLVE_TASK',
        parameter => 'ACCEPT_PLANS',
        value => 'TRUE'
    );
END;
/

-- Monitor plan evolution results
SELECT task_name, execution_start, execution_end, status
FROM DBA_ADVISOR_EXECUTIONS
WHERE task_name = 'SYS_AUTO_SPM_EVOLVE_TASK';
```

## 6. Blockchain ve Immutable Tables

### Blockchain Tables (21c+)

```sql
-- Create blockchain table
CREATE BLOCKCHAIN TABLE audit_blockchain (
    transaction_id NUMBER,
    user_id NUMBER,
    action_type VARCHAR2(50),
    table_name VARCHAR2(100),
    record_id NUMBER,
    old_values CLOB,
    new_values CLOB,
    transaction_timestamp TIMESTAMP DEFAULT SYSTIMESTAMP,
    CONSTRAINT audit_blockchain_pk PRIMARY KEY (transaction_id)
)
NO DROP UNTIL 365 DAYS IDLE
NO DELETE LOCKED
HASHING USING "SHA2_512" VERSION "v1";

-- Insert audit records (cannot be modified or deleted)
INSERT INTO audit_blockchain VALUES (
    audit_seq.NEXTVAL, 100, 'UPDATE', 'EMPLOYEES', 1001,
    '{"salary": 5000}', '{"salary": 5500}', SYSTIMESTAMP
);

-- Verify blockchain integrity
SELECT COUNT(*) as total_records,
       BLOCKCHAIN_TABLE_VALIDATE('AUDIT_BLOCKCHAIN') as is_valid
FROM audit_blockchain;

-- View blockchain metadata
SELECT chain_id, chain_sequence, creation_time, hash
FROM USER_BLOCKCHAIN_TABLES
WHERE table_name = 'AUDIT_BLOCKCHAIN';
```

### Immutable Tables (21c+)

```sql
-- Create immutable table
CREATE IMMUTABLE TABLE financial_records (
    record_id NUMBER,
    account_id NUMBER,
    transaction_amount NUMBER(15,2),
    transaction_date DATE,
    description VARCHAR2(500),
    created_timestamp TIMESTAMP DEFAULT SYSTIMESTAMP,
    CONSTRAINT financial_records_pk PRIMARY KEY (record_id)
)
NO DROP UNTIL 2555 DAYS IDLE
NO DELETE UNTIL 2555 DAYS AFTER INSERT;

-- Data can only be inserted, never updated or deleted
INSERT INTO financial_records VALUES (
    1, 12345, 1500.00, SYSDATE, 'Salary deposit', SYSTIMESTAMP
);

-- Attempt to update will fail
-- UPDATE financial_records SET transaction_amount = 1600 WHERE record_id = 1;
-- ORA-05715: operation not allowed on the Immutable Table

-- View immutable table properties
SELECT table_name, drop_lockdown_type, delete_lockdown_type
FROM USER_IMMUTABLE_TABLES;
```

## 7. JavaScript in Database (21c+)

### MLE (Multilingual Engine) Support

```sql
-- Enable JavaScript environment
CREATE MLE ENV js_env LANGUAGE JAVASCRIPT;

-- Create JavaScript module
CREATE OR REPLACE MLE MODULE math_utils LANGUAGE JAVASCRIPT AS
$$
function calculateCompoundInterest(principal, rate, time, frequency) {
    return principal * Math.pow((1 + rate / frequency), frequency * time);
}

function formatCurrency(amount, currency = 'USD') {
    return new Intl.NumberFormat('en-US', {
        style: 'currency',
        currency: currency
    }).format(amount);
}

function validateEmail(email) {
    const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return regex.test(email);
}

exports.calculateCompoundInterest = calculateCompoundInterest;
exports.formatCurrency = formatCurrency;
exports.validateEmail = validateEmail;
$$;

-- Create PL/SQL wrapper for JavaScript functions
CREATE OR REPLACE FUNCTION calculate_interest(
    p_principal NUMBER,
    p_rate NUMBER,
    p_time NUMBER,
    p_frequency NUMBER DEFAULT 12
) RETURN NUMBER AS MLE MODULE math_utils SIGNATURE 'calculateCompoundInterest(number, number, number, number)';

CREATE OR REPLACE FUNCTION format_currency(
    p_amount NUMBER,
    p_currency VARCHAR2 DEFAULT 'USD'
) RETURN VARCHAR2 AS MLE MODULE math_utils SIGNATURE 'formatCurrency(number, string)';

CREATE OR REPLACE FUNCTION validate_email(p_email VARCHAR2)
RETURN NUMBER AS MLE MODULE math_utils SIGNATURE 'validateEmail(string)';

-- Usage examples
SELECT employee_id,
       salary,
       calculate_interest(salary, 0.05, 10, 12) as retirement_projection,
       format_currency(salary, 'EUR') as formatted_salary
FROM employees
WHERE validate_email(email) = 1;
```

### Advanced JavaScript Integration

```sql
-- Complex data processing with JavaScript
CREATE OR REPLACE MLE MODULE data_processor LANGUAGE JAVASCRIPT AS
$$
function processEmployeeData(employees) {
    return employees.map(emp => {
        const yearsOfService = new Date().getFullYear() - new Date(emp.hire_date).getFullYear();
        const performanceBonus = emp.salary * 0.1 * Math.min(yearsOfService / 10, 1);

        return {
            ...emp,
            years_of_service: yearsOfService,
            performance_bonus: Math.round(performanceBonus * 100) / 100,
            total_compensation: emp.salary + performanceBonus,
            category: emp.salary > 50000 ? 'Senior' : 'Junior'
        };
    });
}

function generateReport(data) {
    const summary = data.reduce((acc, emp) => {
        acc.totalSalary += emp.salary;
        acc.totalBonus += emp.performance_bonus;
        acc.count += 1;
        acc.avgYears += emp.years_of_service;
        return acc;
    }, { totalSalary: 0, totalBonus: 0, count: 0, avgYears: 0 });

    summary.avgSalary = summary.totalSalary / summary.count;
    summary.avgYears = summary.avgYears / summary.count;

    return summary;
}

exports.processEmployeeData = processEmployeeData;
exports.generateReport = generateReport;
$$;

-- PL/SQL integration
CREATE OR REPLACE FUNCTION process_employee_data(p_dept_id NUMBER)
RETURN CLOB IS
    v_employees_json CLOB;
    v_processed_json CLOB;
BEGIN
    -- Get employee data as JSON
    SELECT JSON_ARRAYAGG(
        JSON_OBJECT(
            'employee_id' VALUE employee_id,
            'name' VALUE first_name || ' ' || last_name,
            'salary' VALUE salary,
            'hire_date' VALUE TO_CHAR(hire_date, 'YYYY-MM-DD')
        )
    ) INTO v_employees_json
    FROM employees
    WHERE department_id = p_dept_id;

    -- Process with JavaScript
    SELECT data_processor.processEmployeeData(JSON_ARRAY(v_employees_json))
    INTO v_processed_json
    FROM dual;

    RETURN v_processed_json;
END;
/
```

## 8. SQL Macros (19c+)

### Scalar SQL Macros

```sql
-- Create scalar SQL macro
CREATE OR REPLACE FUNCTION get_employee_grade(p_salary NUMBER)
RETURN VARCHAR2 SQL_MACRO(SCALAR) IS
BEGIN
    RETURN q'{
        CASE
            WHEN p_salary > 100000 THEN 'Executive'
            WHEN p_salary > 75000 THEN 'Senior'
            WHEN p_salary > 50000 THEN 'Mid-level'
            WHEN p_salary > 25000 THEN 'Junior'
            ELSE 'Entry'
        END
    }';
END;
/

-- Usage in SQL
SELECT employee_id,
       first_name,
       salary,
       get_employee_grade(salary) as grade
FROM employees;

-- Complex calculation macro
CREATE OR REPLACE FUNCTION calculate_tax(
    p_salary NUMBER,
    p_country VARCHAR2 DEFAULT 'TR'
) RETURN NUMBER SQL_MACRO(SCALAR) IS
BEGIN
    RETURN q'{
        CASE p_country
            WHEN 'TR' THEN
                CASE
                    WHEN p_salary > 180000 THEN p_salary * 0.35
                    WHEN p_salary > 98000 THEN p_salary * 0.27
                    WHEN p_salary > 34000 THEN p_salary * 0.20
                    ELSE p_salary * 0.15
                END
            WHEN 'US' THEN
                CASE
                    WHEN p_salary > 518400 THEN p_salary * 0.37
                    WHEN p_salary > 207350 THEN p_salary * 0.35
                    WHEN p_salary > 86375 THEN p_salary * 0.24
                    ELSE p_salary * 0.22
                END
            ELSE p_salary * 0.20
        END
    }';
END;
/
```

### Table SQL Macros

```sql
-- Create table SQL macro for dynamic filtering
CREATE OR REPLACE FUNCTION get_filtered_employees(
    p_department VARCHAR2 DEFAULT NULL,
    p_min_salary NUMBER DEFAULT 0,
    p_hire_year NUMBER DEFAULT NULL
) RETURN VARCHAR2 SQL_MACRO(TABLE) IS
BEGIN
    RETURN q'{
        SELECT e.employee_id,
               e.first_name,
               e.last_name,
               e.email,
               e.salary,
               e.hire_date,
               d.department_name
        FROM employees e
        JOIN departments d ON e.department_id = d.department_id
        WHERE (p_department IS NULL OR d.department_name = p_department)
        AND e.salary >= p_min_salary
        AND (p_hire_year IS NULL OR EXTRACT(YEAR FROM e.hire_date) = p_hire_year)
    }';
END;
/

-- Usage
SELECT * FROM get_filtered_employees('IT', 50000, 2020);
SELECT * FROM get_filtered_employees(p_min_salary => 75000);

-- Complex analytical macro
CREATE OR REPLACE FUNCTION employee_analytics(p_dept_id NUMBER)
RETURN VARCHAR2 SQL_MACRO(TABLE) IS
BEGIN
    RETURN q'{
        WITH dept_stats AS (
            SELECT department_id,
                   COUNT(*) as emp_count,
                   AVG(salary) as avg_salary,
                   STDDEV(salary) as salary_stddev,
                   MIN(hire_date) as oldest_hire,
                   MAX(hire_date) as newest_hire
            FROM employees
            WHERE (p_dept_id IS NULL OR department_id = p_dept_id)
            GROUP BY department_id
        ),
        emp_with_stats AS (
            SELECT e.*,
                   ds.avg_salary,
                   ds.salary_stddev,
                   (e.salary - ds.avg_salary) / NULLIF(ds.salary_stddev, 0) as salary_zscore,
                   MONTHS_BETWEEN(SYSDATE, e.hire_date) as tenure_months
            FROM employees e
            JOIN dept_stats ds ON e.department_id = ds.department_id
            WHERE (p_dept_id IS NULL OR e.department_id = p_dept_id)
        )
        SELECT employee_id,
               first_name,
               last_name,
               salary,
               ROUND(avg_salary, 2) as dept_avg_salary,
               ROUND(salary_zscore, 2) as salary_zscore,
               ROUND(tenure_months/12, 1) as tenure_years,
               CASE
                   WHEN salary_zscore > 1.5 THEN 'High Performer'
                   WHEN salary_zscore > 0.5 THEN 'Above Average'
                   WHEN salary_zscore > -0.5 THEN 'Average'
                   ELSE 'Below Average'
               END as performance_category
        FROM emp_with_stats
    }';
END;
/
```

## 9. Private Temporary Tables (18c+)

```sql
-- Create private temporary table
CREATE PRIVATE TEMPORARY TABLE ora$ptt_session_calc (
    calc_id NUMBER,
    operation VARCHAR2(50),
    operand1 NUMBER,
    operand2 NUMBER,
    result NUMBER,
    calc_time TIMESTAMP DEFAULT SYSTIMESTAMP
) ON COMMIT PRESERVE DEFINITION;

-- Use for session-specific calculations
INSERT INTO ora$ptt_session_calc VALUES (1, 'ADD', 10, 20, 30, SYSTIMESTAMP);
INSERT INTO ora$ptt_session_calc VALUES (2, 'MULTIPLY', 5, 6, 30, SYSTIMESTAMP);

-- Query session data
SELECT operation, COUNT(*) as operation_count
FROM ora$ptt_session_calc
GROUP BY operation;

-- Complex session processing
CREATE PRIVATE TEMPORARY TABLE ora$ptt_employee_processing (
    employee_id NUMBER,
    processing_step VARCHAR2(100),
    step_result CLOB,
    processing_time TIMESTAMP,
    step_order NUMBER
) ON COMMIT DROP DEFINITION;

-- Multi-step processing procedure
CREATE OR REPLACE PROCEDURE process_employee_batch(p_dept_id NUMBER) IS
    v_step_order NUMBER := 1;
BEGIN
    -- Step 1: Data validation
    INSERT INTO ora$ptt_employee_processing
    SELECT employee_id, 'VALIDATION',
           CASE
               WHEN email IS NULL THEN 'Missing email'
               WHEN salary <= 0 THEN 'Invalid salary'
               ELSE 'Valid'
           END,
           SYSTIMESTAMP, v_step_order
    FROM employees
    WHERE department_id = p_dept_id;

    v_step_order := v_step_order + 1;

    -- Step 2: Salary calculation
    INSERT INTO ora$ptt_employee_processing
    SELECT employee_id, 'SALARY_CALC',
           JSON_OBJECT(
               'current_salary' VALUE salary,
               'annual_increase' VALUE salary * 0.05,
               'new_salary' VALUE salary * 1.05
           ),
           SYSTIMESTAMP, v_step_order
    FROM employees
    WHERE department_id = p_dept_id;

    -- Final processing summary
    SELECT processing_step, COUNT(*) as step_count
    FROM ora$ptt_employee_processing
    GROUP BY processing_step, step_order
    ORDER BY step_order;
END;
/
```

## 10. Vector Search (23c)

### Vector Data Types

```sql
-- Create table with vector columns
CREATE TABLE document_embeddings (
    doc_id NUMBER PRIMARY KEY,
    document_title VARCHAR2(200),
    document_content CLOB,
    content_vector VECTOR(1536, FLOAT32),
    summary_vector VECTOR(768, FLOAT64),
    created_date DATE DEFAULT SYSDATE
);

-- Create vector index for similarity search
CREATE VECTOR INDEX idx_content_vector ON document_embeddings (content_vector)
PARAMETERS ('type HNSW, metric COSINE, efConstruction 300, M 16');

-- Insert sample vectors (normally generated by ML models)
INSERT INTO document_embeddings VALUES (
    1,
    'Database Performance Tuning',
    'This document covers various techniques for optimizing database performance...',
    VECTOR('[0.1, 0.2, 0.3, 0.4, ...]', 1536, FLOAT32),
    VECTOR('[0.5, 0.6, 0.7, ...]', 768, FLOAT64),
    SYSDATE
);
```

### Vector Similarity Search

```sql
-- Find similar documents using vector search
SELECT doc_id,
       document_title,
       VECTOR_DISTANCE(content_vector, :query_vector, COSINE) as similarity_score
FROM document_embeddings
ORDER BY VECTOR_DISTANCE(content_vector, :query_vector, COSINE)
FETCH FIRST 10 ROWS ONLY;

-- Hybrid search combining vector and text
WITH vector_results AS (
    SELECT doc_id, document_title,
           VECTOR_DISTANCE(content_vector, :query_vector, COSINE) as vector_score
    FROM document_embeddings
    ORDER BY VECTOR_DISTANCE(content_vector, :query_vector, COSINE)
    FETCH FIRST 50 ROWS ONLY
),
text_results AS (
    SELECT doc_id, document_title,
           SCORE(1) as text_score
    FROM document_embeddings
    WHERE CONTAINS(document_content, :query_text, 1) > 0
)
SELECT v.doc_id, v.document_title,
       v.vector_score,
       NVL(t.text_score, 0) as text_score,
       (v.vector_score * 0.7 + NVL(t.text_score, 0) * 0.3) as combined_score
FROM vector_results v
LEFT JOIN text_results t ON v.doc_id = t.doc_id
ORDER BY combined_score;
```

### Vector Analytics

```sql
-- Cluster analysis using vectors
SELECT doc_id,
       document_title,
       CLUSTER_ID(vector_clustering_model USING content_vector) as cluster_id,
       CLUSTER_PROBABILITY(vector_clustering_model USING content_vector) as cluster_prob
FROM document_embeddings;

-- Vector aggregation for category analysis
SELECT category,
       VECTOR_SERIALIZE(
           VECTOR_NORM(
               VECTOR_SUM(content_vector)
           )
       ) as category_centroid
FROM document_embeddings
GROUP BY category;

-- Anomaly detection using vector distances
WITH avg_vectors AS (
    SELECT AVG(content_vector) as mean_vector
    FROM document_embeddings
)
SELECT d.doc_id,
       d.document_title,
       VECTOR_DISTANCE(d.content_vector, a.mean_vector, EUCLIDEAN) as distance_from_mean,
       CASE
           WHEN VECTOR_DISTANCE(d.content_vector, a.mean_vector, EUCLIDEAN) > 2.0
           THEN 'ANOMALY'
           ELSE 'NORMAL'
       END as classification
FROM document_embeddings d
CROSS JOIN avg_vectors a
ORDER BY distance_from_mean DESC;
```

## Modern Development Best Practices

### 1. Version-Aware Development

```sql
-- Check Oracle version and features
SELECT banner, version, version_legacy
FROM v$version
WHERE banner LIKE 'Oracle Database%';

-- Feature availability check
SELECT name, value, description
FROM v$system_parameter
WHERE name IN ('compatible', 'enable_automatic_maintenance_tasks');

-- Conditional compilation for version compatibility
CREATE OR REPLACE PACKAGE version_aware_pkg IS
    FUNCTION get_json_data(p_id NUMBER) RETURN CLOB;
END;
/

CREATE OR REPLACE PACKAGE BODY version_aware_pkg IS
    FUNCTION get_json_data(p_id NUMBER) RETURN CLOB IS
        v_result CLOB;
    BEGIN
        $IF DBMS_DB_VERSION.VERSION >= 21 $THEN
            -- Use native JSON binary format for 21c+
            SELECT JSON_SERIALIZE(data FORMAT JSON)
            INTO v_result
            FROM json_table_21c WHERE id = p_id;
        $ELSE
            -- Fallback for older versions
            SELECT JSON_QUERY(data, '$')
            INTO v_result
            FROM json_table_legacy WHERE id = p_id;
        $END

        RETURN v_result;
    END;
END;
/
```

### 2. Cloud-Ready Development

```sql
-- Cloud-aware configuration
CREATE OR REPLACE PACKAGE cloud_config IS
    FUNCTION is_autonomous_database RETURN BOOLEAN;
    FUNCTION get_service_name RETURN VARCHAR2;
    PROCEDURE setup_cloud_features;
END;
/

CREATE OR REPLACE PACKAGE BODY cloud_config IS
    FUNCTION is_autonomous_database RETURN BOOLEAN IS
        v_count NUMBER;
    BEGIN
        SELECT COUNT(*) INTO v_count
        FROM v$parameter
        WHERE name = 'cloud_service' AND value IS NOT NULL;

        RETURN v_count > 0;
    END;

    FUNCTION get_service_name RETURN VARCHAR2 IS
        v_service VARCHAR2(100);
    BEGIN
        SELECT value INTO v_service
        FROM v$parameter
        WHERE name = 'service_names'
        AND rownum = 1;

        RETURN v_service;
    END;

    PROCEDURE setup_cloud_features IS
    BEGIN
        IF is_autonomous_database THEN
            -- Enable cloud-specific features
            EXECUTE IMMEDIATE 'ALTER SESSION SET "_enable_automatic_maintenance" = TRUE';
            DBMS_OUTPUT.PUT_LINE('Autonomous Database features enabled');
        END IF;
    END;
END;
/
```

**Sonraki Bölümde:** Development Environment ve Tools setup'ını detaylandıracağım.
