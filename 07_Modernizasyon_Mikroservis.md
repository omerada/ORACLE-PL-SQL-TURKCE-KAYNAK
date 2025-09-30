# Oracle Forms'tan Java Spring Microservices Modernizasyonu

## 1. Modernizasyon Neden Gerekli?

### Oracle Forms'un Sınırlamaları

- **Platform Bağımlılığı**: Windows/Java client gereksinimi
- **Scalability**: Sınırlı ölçeklenebilirlik
- **Modern UI/UX**: Eski kullanıcı arayüzü
- **Integration**: REST API desteği eksikliği
- **Mobile Support**: Mobil uyumluluk yok
- **Cloud**: Bulut desteği sınırlı
- **Maintenance Cost**: Yüksek lisans ve bakım maliyeti

### Mikroservis Mimarisinin Avantajları

- **Scalability**: Bağımsız ölçeklendirme
- **Technology Diversity**: Farklı teknolojiler kullanabilme
- **Resilience**: Hata izolasyonu
- **Deployment**: Bağımsız deployment
- **Team Organization**: DevOps ve agile uyum

## 2. Dönüşüm Stratejisi

### 1. Strangler Fig Pattern

Eski sistemi kademeli olarak değiştirme:

```
Legacy Oracle Forms ─────► Gradually Replace ─────► Spring Microservices
       │                                                      │
       │  ┌─ New Feature 1 ────────────── Microservice A     │
       │  ├─ New Feature 2 ────────────── Microservice B     │
       │  ├─ Migrated Feature 3 ──────── Microservice C     │
       │  └─ Legacy Features ─────────── (To be migrated)   │
       │                                                      │
    Database ──────────────────────────── Shared/Decomposed ──
```

### 2. Big Bang vs Incremental

**Önerilen: Incremental Approach**

- Module bazında migration
- Risk dağılımı
- Continuous delivery
- User feedback incorporation

## 3. Forms → Spring Boot Mapping

### Form Yapısından Mikroservis Mimarisine

```
Oracle Forms Architecture          Spring Boot Microservices
═══════════════════════════      ═════════════════════════════

FORM                         →    Spring Boot Application
├── BLOCK (Data)            →    ├── Entity/Repository Layer
├── TRIGGER (Business Logic) →    ├── Service Layer
├── CANVAS (UI)             →    ├── Controller Layer (REST API)
└── WINDOW (Navigation)     →    └── Frontend (React/Angular)

Form-Level Variables        →    Application Properties/Context
Global Variables           →    Shared Configuration Service
Master-Detail Relations    →    JPA Entity Relationships
Built-in Validations      →    Bean Validation (@Valid)
PL/SQL Packages          →    Spring Services/Components
```

## 4. Teknik Mapping Detayları

### Database Access Transformation

**Oracle Forms Approach:**

```sql
-- Forms'ta direct SQL
DECLARE
    CURSOR emp_cursor IS
        SELECT emp_id, first_name, salary FROM employees;
BEGIN
    FOR emp_rec IN emp_cursor LOOP
        :BLOCK.EMP_ID := emp_rec.emp_id;
        -- Process record
    END LOOP;
END;
```

**Spring Boot Equivalent:**

```java
// Entity
@Entity
@Table(name = "employees")
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "emp_seq")
    private Long empId;

    @Column(name = "first_name")
    private String firstName;

    private BigDecimal salary;
    // getters/setters
}

// Repository
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    @Query("SELECT e FROM Employee e WHERE e.department.id = :deptId")
    List<Employee> findByDepartmentId(@Param("deptId") Long deptId);
}

// Service
@Service
public class EmployeeService {
    @Autowired
    private EmployeeRepository employeeRepository;

    public List<EmployeeDto> getEmployeesByDepartment(Long deptId) {
        return employeeRepository.findByDepartmentId(deptId)
                .stream()
                .map(this::convertToDto)
                .collect(Collectors.toList());
    }
}

// Controller
@RestController
@RequestMapping("/api/employees")
public class EmployeeController {
    @Autowired
    private EmployeeService employeeService;

    @GetMapping("/department/{deptId}")
    public ResponseEntity<List<EmployeeDto>> getByDepartment(@PathVariable Long deptId) {
        return ResponseEntity.ok(employeeService.getEmployeesByDepartment(deptId));
    }
}
```

### Business Logic Migration

**Forms Trigger Logic:**

```sql
-- PRE-INSERT trigger
BEGIN
    SELECT employee_seq.NEXTVAL INTO :EMPLOYEE.EMP_ID FROM DUAL;
    :EMPLOYEE.HIRE_DATE := SYSDATE;

    -- Validation
    IF :EMPLOYEE.SALARY < 1000 THEN
        MESSAGE('Maaş minimum 1000 olmalı');
        RAISE FORM_TRIGGER_FAILURE;
    END IF;

    -- Business rule
    IF :EMPLOYEE.JOB_ID = 'MANAGER' THEN
        :EMPLOYEE.SALARY := :EMPLOYEE.SALARY * 1.2;
    END IF;
END;
```

**Spring Boot Service:**

```java
@Service
@Transactional
public class EmployeeService {

    @Autowired
    private EmployeeRepository employeeRepository;

    @Autowired
    private SequenceService sequenceService;

    public EmployeeDto createEmployee(CreateEmployeeDto createDto) {
        // Validation
        validateEmployee(createDto);

        Employee employee = new Employee();
        employee.setEmpId(sequenceService.getNextEmployeeId());
        employee.setFirstName(createDto.getFirstName());
        employee.setHireDate(LocalDate.now());
        employee.setSalary(createDto.getSalary());

        // Business rule
        if ("MANAGER".equals(createDto.getJobId())) {
            employee.setSalary(employee.getSalary().multiply(new BigDecimal("1.2")));
        }

        Employee saved = employeeRepository.save(employee);
        return convertToDto(saved);
    }

    private void validateEmployee(CreateEmployeeDto dto) {
        if (dto.getSalary().compareTo(new BigDecimal("1000")) < 0) {
            throw new BusinessException("Maaş minimum 1000 olmalı");
        }
    }
}
```

## 5. Mikroservis Decomposition Stratejileri

### 1. Domain-Driven Design (DDD)

```
Monolithic Oracle Forms          Microservices Decomposition
═══════════════════════        ═══════════════════════════════

Employee Management Form   →    ├── Employee Service
├── Employee Data         →    │   ├── Employee CRUD
├── Department Data       →    │   ├── Employee Search
├── Salary Management     →    │   └── Employee Validation
├── Reports              →    │
                         →    ├── Department Service
                         →    │   ├── Department CRUD
                         →    │   └── Department Hierarchy
                         →    │
                         →    ├── Payroll Service
                         →    │   ├── Salary Calculation
                         →    │   ├── Bonus Processing
                         →    │   └── Tax Calculation
                         →    │
                         →    └── Reporting Service
                         →        ├── Report Generation
                         →        └── Data Aggregation
```

### 2. Bounded Context Identification

```java
// Employee Bounded Context
@RestController
@RequestMapping("/api/employees")
public class EmployeeController {
    // Employee-specific operations
}

// Payroll Bounded Context
@RestController
@RequestMapping("/api/payroll")
public class PayrollController {
    // Salary, bonus, tax operations
}

// Department Bounded Context
@RestController
@RequestMapping("/api/departments")
public class DepartmentController {
    // Organizational structure
}
```

## 6. Data Management Strategies

### 1. Database Decomposition

**Shared Database (Initial):**

```
┌─────────────────────────────────────┐
│           Oracle Database           │
├─────────────────────────────────────┤
│  Employee Service  │  Dept Service  │
│  Payroll Service   │  Report Service │
└─────────────────────────────────────┘
```

**Database per Service (Target):**

```
┌───────────────┐  ┌──────────────┐  ┌─────────────┐
│ Employee DB   │  │ Payroll DB   │  │ Dept DB     │
├───────────────┤  ├──────────────┤  ├─────────────┤
│Employee       │  │Payroll       │  │Department   │
│Service        │  │Service       │  │Service      │
└───────────────┘  └──────────────┘  └─────────────┘
```

### 2. Data Consistency Patterns

**Saga Pattern Implementation:**

```java
@Service
public class EmployeeOnboardingSaga {

    @Autowired
    private EmployeeService employeeService;

    @Autowired
    private PayrollService payrollService;

    @Autowired
    private ITService itService;

    public void onboardEmployee(OnboardingDto dto) {
        try {
            // Step 1: Create employee
            EmployeeDto employee = employeeService.createEmployee(dto);

            // Step 2: Setup payroll
            payrollService.setupPayroll(employee.getId(), dto.getSalary());

            // Step 3: Create IT accounts
            itService.createAccounts(employee.getId(), dto.getEmail());

        } catch (Exception e) {
            // Compensating transactions
            compensateOnboarding(dto);
            throw e;
        }
    }

    private void compensateOnboarding(OnboardingDto dto) {
        // Rollback operations
    }
}
```

## 7. REST API Design

### Richardson Maturity Model Level 3

```java
@RestController
@RequestMapping("/api/employees")
public class EmployeeController {

    // GET /api/employees
    @GetMapping
    public ResponseEntity<PagedResponse<EmployeeDto>> getEmployees(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String search) {

        Page<EmployeeDto> employees = employeeService.getEmployees(page, size, search);
        return ResponseEntity.ok(new PagedResponse<>(employees));
    }

    // GET /api/employees/{id}
    @GetMapping("/{id}")
    public ResponseEntity<EmployeeDto> getEmployee(@PathVariable Long id) {
        return employeeService.getEmployee(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    // POST /api/employees
    @PostMapping
    public ResponseEntity<EmployeeDto> createEmployee(
            @Valid @RequestBody CreateEmployeeDto createDto) {

        EmployeeDto created = employeeService.createEmployee(createDto);
        URI location = URI.create("/api/employees/" + created.getId());
        return ResponseEntity.created(location).body(created);
    }

    // PUT /api/employees/{id}
    @PutMapping("/{id}")
    public ResponseEntity<EmployeeDto> updateEmployee(
            @PathVariable Long id,
            @Valid @RequestBody UpdateEmployeeDto updateDto) {

        return employeeService.updateEmployee(id, updateDto)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    // DELETE /api/employees/{id}
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteEmployee(@PathVariable Long id) {
        boolean deleted = employeeService.deleteEmployee(id);
        return deleted ? ResponseEntity.noContent().build() :
                        ResponseEntity.notFound().build();
    }
}
```

## 8. Security Transformation

### Forms Session → JWT + OAuth2

**Oracle Forms Session:**

```sql
-- Forms'ta user validation
BEGIN
    IF :GLOBAL.USER_NAME IS NULL THEN
        GO_FORM('LOGIN_FORM');
    END IF;
END;
```

**Spring Security Configuration:**

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/employees/**").hasRole("USER")
                .requestMatchers(HttpMethod.POST, "/api/employees/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer().jwt();

        return http.build();
    }
}

@RestController
@RequestMapping("/api/employees")
@PreAuthorize("hasRole('HR_MANAGER')")
public class EmployeeController {

    @PostMapping
    @PreAuthorize("hasPermission(#createDto, 'CREATE_EMPLOYEE')")
    public ResponseEntity<EmployeeDto> createEmployee(@RequestBody CreateEmployeeDto createDto) {
        // Implementation
    }
}
```

## 9. Frontend Modernization

### Forms UI → React/Angular

**Forms Canvas Elements → React Components:**

```jsx
// Employee Form Component
const EmployeeForm = () => {
  const [employee, setEmployee] = useState({});
  const [errors, setErrors] = useState({});

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      await employeeService.createEmployee(employee);
      navigate("/employees");
    } catch (error) {
      setErrors(error.response.data.errors);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <div className="form-group">
        <label>First Name</label>
        <input
          type="text"
          value={employee.firstName || ""}
          onChange={(e) =>
            setEmployee({ ...employee, firstName: e.target.value })
          }
          className={errors.firstName ? "error" : ""}
        />
        {errors.firstName && (
          <span className="error-text">{errors.firstName}</span>
        )}
      </div>

      <div className="form-group">
        <label>Salary</label>
        <input
          type="number"
          value={employee.salary || ""}
          onChange={(e) => setEmployee({ ...employee, salary: e.target.value })}
          min="1000"
        />
      </div>

      <button type="submit">Save Employee</button>
    </form>
  );
};
```

## 10. Monitoring ve Observability

### Forms'tan Modern Observability'e

```java
// Metrics
@RestController
public class EmployeeController {

    private final MeterRegistry meterRegistry;
    private final Counter employeeCreatedCounter;

    public EmployeeController(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.employeeCreatedCounter = Counter.builder("employees.created")
                .description("Number of employees created")
                .register(meterRegistry);
    }

    @PostMapping
    @Timed(name = "employee.creation.time", description = "Time taken to create employee")
    public ResponseEntity<EmployeeDto> createEmployee(@RequestBody CreateEmployeeDto dto) {
        EmployeeDto created = employeeService.createEmployee(dto);
        employeeCreatedCounter.increment();
        return ResponseEntity.ok(created);
    }
}

// Distributed Tracing
@Service
public class EmployeeService {

    @NewSpan("employee-validation")
    public void validateEmployee(@SpanTag("employeeId") Long id) {
        // Validation logic
    }
}

// Health Checks
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        try {
            employeeRepository.count();
            return Health.up().withDetail("database", "Available").build();
        } catch (Exception e) {
            return Health.down().withDetail("database", "Unavailable").build();
        }
    }
}
```

## 11. Migration Checklist

### Pre-Migration Analysis

- [ ] Forms inventory and dependency mapping
- [ ] Database schema analysis
- [ ] Business rules extraction
- [ ] User workflow documentation
- [ ] Performance requirements definition

### Technical Implementation

- [ ] Microservices boundary definition
- [ ] API contract design
- [ ] Database migration strategy
- [ ] Security implementation
- [ ] Frontend component development

### Testing Strategy

- [ ] Unit tests for business logic
- [ ] Integration tests for APIs
- [ ] End-to-end user journey tests
- [ ] Performance testing
- [ ] Security testing

### Deployment & Operations

- [ ] CI/CD pipeline setup
- [ ] Monitoring and alerting
- [ ] Logging strategy
- [ ] Backup and disaster recovery
- [ ] Documentation and training

## Pratik Uygulama Önerileri

1. **Pilot Project**: Küçük, bağımsız bir modül ile başlayın
2. **Incremental Migration**: Büyük bang yerine kademeli geçiş
3. **API First**: Önce API'leri tasarlayın, sonra implementasyon
4. **Testing Strategy**: Comprehensive test coverage
5. **Team Training**: Spring Boot ve mikroservis eğitimleri
6. **Monitoring**: Day-1'den observability

**Sonraki Bölümde:** Proje özetleri ve öğrenme yol haritası sunacağız.
