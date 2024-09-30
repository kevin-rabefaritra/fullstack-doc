# Hexagonal architecture

## Key points
- Create loosely coupled components that can be easily connected.
- Is said to be the origin of microservices architecture.
- Increases code complexity and performance due to extra layers.

## Classic architecture
Starting from a classic *Presentation-Logic-Data* architecture:
- users interact with the **presentation layer**
- **business logic** is performed in the backend
- **data** is manipulated by the logic layer

```
Presentation <-> Business logic <-> Data
```

For example, say we have an app to manage Employees, we will have the following classes:
- `Employee` entity class.
- `EmployeeController` class for exposing API endpoints.
- `EmployeeService` class for our app logic.
- `EmployeeRepository` class for database operations.

Our app is straightforward and components are **tightly coupled**, technology decisions need to be made from the beginning.

If we want to replace the data layer (eg. use an API-based database), we need to rewrite the `EmployeeRepository` class and possibly update the `EmployeeService` class.

## "Classic" decoupling
We can increase decoupling with abstraction.
- We replace the `EmployeeService` class to an **interface**.
- We replace the `EmployeeRepository` class to an **interface**.

```
Presentation <-> Interface <-> Business logic <-> Interface <-> Data
```

This way, the presentation layer is less concerned about the business logic details, so is the business logic regarding the data layer.

```
interface EmployeeRepository {
  void save(Employee employee);
  void deleteById(UUID id);
}
```

```
class InMemoryEmployeeRepository implements EmployeeRepository {
  Map<UUID, Employee> employees = new HashMap<>();

  @Override
  Employee save(Employee employee) {
    return employees.put(UUID.randomUUID(), employee);
  }

  @Override
  void deleteById(UUID id) {
    employees.remove(id);
  }
}
```

```
class DynamoDBEmployeeRepository implements EmployeeRepository {
  
  @Override
  Employee save(Employee employee) {
    // Implementation using DynamoDB
  }

  @Override
  void deleteById(UUID id) {
    // Implementation using DynamoDB
  }
}
```

The implementation of `EmployeeService` now deals with the interface `EmployeeRepository`.

```
interface EmployeeService {
  Employee hire(Employee employee);
  void fire(Employee employee);
}
```

```
class EmployeeServiceImpl implements EmployeeService {

  final EmployeeRepository employeeRepository;

  EmployeeServiceImpl(EmployeeRepository employeeRepository) {
    this.employeeRepository = employeeRepository;
  }

  @Override
  Employee hire(Employee employee) {
    // some app logic ..
    return this.employeeRepository.save(employee);
  }

  @Override
  void fire(UUID id) {
    // some app logic
    this.employeeRepository.deleteById(id);
  }
}
```

If we want to replace our data layer, we just need to replace the `EmployeeRepository` bean returned instance.

We can also take advantage of Spring **profiles** to have the instance based on the configuration profile.

```
@Bean
EmployeeRepository employeeRepository() {
  /*
  * return new InMemoryEmployeeRepository();
  * or
  * return new DynamoDBEmployeeRepository();
  */
}
```

## Hexagonal approach
We take decoupling one step further with **ports & adapters**. 

Suppose human resources managers are in charge of hiring and firing employees. We introduce two ports: **HiringEmployee** and **FiringEmployee**.

```
                  HiringEmployee
Presentation <->                  <-> HumanResources <-> Employees (data)
                  FiringEmployee
```

Here, `HiringEmployee`, `FiringEmployee`, `Employees` are interfaces.

```
interface HiringEmployee {
  Employee hire(Employee employee);
}
```

```
interface FiringEmployee {
  void fire(UUID id);
}
```

```
interface Employees {
  Employee save(Employee employee);
  void deleteById(UUID id);
}
```

The class `HumanResources` operates entities and repositories.

```
class HumanResources implements HiringEmployee, FiringEmployee {

  final Employees employees;

  @Override
  Employee hire(Employee employee) {
    // Some app logic
    return this.employees.save(employee);
  }

  @Override
  void fire(UUID id) {
    // Some app logic
    this.employees.deleteById(id);
  }
}
```

Acceptance tests will deal less with technical concerns in favor of business requirements.

```

@BeforeEach
void setup() {
  employees = InMemoryEmployees();
  humanResources = new HumanResources(employees);
}

@Test
void testEmployeeHasBeenHired() {
  Employee employee = new Employee(...);
  Employee savedEmployee = humanResources.hire(employee);

  assertNotNull(savedEmployee.getId());
}
```

Our presentation layer will look like this 

```
@RestController
@RequestMapping
class EmployeeController {

  final HiringEmployee hiringEmployee;
  final FiringEmployee firingEmployee;

  @PostMapping("...")
  HiringResponse hire(@RequestBody HiringRequest request) {
    Employee employee = this.hiringEmployee.hire(request.employee());
    return new HiringResponse(employee);
  }
}
```

Note: `HiringResponse` and `HiringRequest` omitted for brevity.