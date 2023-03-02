## Spring Boot - Spring Data JPA
>參考網站：https://morosedog.gitlab.io/springboot-20190328-springboot14/
>
>而在實際的專案開發中，最基本的資料庫操作不外乎「CRUD」，而這些操作除了資料表名稱和結構不同外，其語法都是類似的，開發人員需要寫大量類似而枯燥的語法來完成業務邏輯
>
>為了解決抽象各個Java實體基本的「CRUD」操作，我們通常會以泛型的方式封裝一個模板Dao來進行抽像簡化，但是這樣依然不是很方便，我們需要針對每個實體編寫一個繼承自泛型模板Dao的接口，再編寫該接口的實現。雖然一些基礎的資料訪問已經可以得到很好的複用，但是在代碼結構上針對每個實體都會有一堆Dao的接口和實現
>
>由於模板Dao的實現，使得這些具體實體的Dao層已經變的非常”薄”，有一些具體實體的Dao實現可能完全就是對模板Dao的簡單代理，並且往往這樣的實現類可能會出現在很多實體上。 Spring-data-jpa的出現正可以讓這樣一個已經很”薄”的資料訪問層變成只是一層接口的編寫方式
>
>**Spring Data JPA是Spring基於Hibernate開發的一個JPA框架。可以極大的簡化JPA的寫法，可以在幾乎不用寫具體代碼的情況下，實現對資料的訪問和操作。除了「CRUD」外，還包括如分頁、排序​​等一些常用的功能**
>>### Spring Data JPA 接口和核心概念
>>![image](https://user-images.githubusercontent.com/101872264/222439928-a07c4847-f823-440a-9690-b42dc838e31e.png)
>>### Spring Data JPA 方法命名效果
>>可以通過方法命名規則進行相關資料庫操作，可以減少很多代碼
>>
>>![image](https://user-images.githubusercontent.com/101872264/222441439-c5482a97-7fde-4201-9c37-58b22ee94635.png)

### 創建實體類別
**Employee.java程式碼，建構實體類別並將相關屬性與資料庫中字段相互映射**
```
@Entity
@Table(name="employee")
public class Employee {

	// define fields
	@Id
	@GeneratedValue(strategy=GenerationType.IDENTITY)
	@Column(name="id")
	private int id;
	
	@Column(name="first_name")
	private String firstName;
	
	@Column(name="last_name")
	private String lastName;
	
	@Column(name="email")
	private String email;
	
	// define constructors
	public Employee() {
		
	}

	public Employee(String firstName, String lastName, String email) {
		this.firstName = firstName;
		this.lastName = lastName;
		this.email = email;
	}
	
	// define getter/setter
	以下省略...
}
```
### 創建Spring Data JPA Repository
**建立資料訪問對象(DAO)，這裡即透過擴展JpaRepository來達成，給定我們要的實體類型以及主鍵型別，如此，只需擴展不須任何代碼即可獲得基本CRUD操作方法**
```
public interface EmployeeRepository extends JpaRepository<Employee, Integer> {

	// that's it ... no need to write any code
}
```
### 建立服務層方法接口
```
public interface EmployeeService {

	public List<Employee> findAll();
	
	public Employee findById(int theId);
	
	public void save(Employee theEmployee);
	
	public void daleteById(int theId);
}
```
### 完成服務層接口方法實現
**EmployeeServiceImpl.java程式碼，基本上僅注入EmployeeRepository，並委託調用EmployeeRepository中的方法，與先前調用DAO方法相同，只是現在DAO透過EmployeeRepository完成**
```
@Service
public class EmployeeServiceImpl implements EmployeeService {
	
	private EmployeeRepository employeeRepository;
	
	@Autowired
	public EmployeeServiceImpl(EmployeeRepository theEmployeeRepository) {
		employeeRepository = theEmployeeRepository;
	}

	@Override
	public List<Employee> findAll() {		
		return employeeRepository.findAll();
	}

	@Override
	public Employee findById(int theId) {
		Optional<Employee> result = employeeRepository.findById(theId);
		
		Employee theEmployee = null;
		
		if (result.isPresent()) {
			theEmployee = result.get();
		}
		else {
			// we didn't find the employee
			throw new RuntimeException("Did not find employee id - " + theId);
		}
		return theEmployee;
	}

	@Override
	public void save(Employee theEmployee) {
		employeeRepository.save(theEmployee);
	}

	@Override
	@Transactional
	public void daleteById(int theId) {
		employeeRepository.deleteById(theId);
	}
}
```
### 建立Controller來完成各個方法的映射配置
```
@RestController
@RequestMapping("/api")
public class EmployeeRestController {
	
	private EmployeeService employeeService;
	
	@Autowired
	public EmployeeRestController(EmployeeService theEmployeeService) {
		employeeService = theEmployeeService;
	}
	
	// expose "/employees" and return list of employees
	@GetMapping("/employees")
	public List<Employee> findAll(){
		return employeeService.findAll();
	}
	
	// add mapping for GET /employee/{employeeId}
	@GetMapping("/employees/{employeeId}")
	public Employee getEmployee(@PathVariable int employeeId) {
		Employee theEmployee = employeeService.findById(employeeId);
		
		if (theEmployee == null)
			throw new RuntimeException("Employee id not found - " + employeeId);
		
		return theEmployee;
	}
	
	// add mapping for POST /employees - add new employee
	@PostMapping("/employees")
	public Employee addEmployee(@RequestBody Employee theEmployee) {
		
		// also just in case they pass an id in JSON ... set id to 0
		// this is to force a save of new item ... instead of update
		
		theEmployee.setId(0);
		
		employeeService.save(theEmployee);
		
		return theEmployee;
	}
	
	// add mapping for PUT /employees - update existing employee
	@PutMapping("/employees")
	public Employee updateEmployee(@RequestBody Employee theEmployee) {
		
		employeeService.save(theEmployee);
		
		return theEmployee;
	}
	
	// add mapping for DELETE /employee/{employeeId} - delete employee
	@DeleteMapping("/employees/{employeeId}")
	public String deleteEmployee(@PathVariable int employeeId) {
		
		Employee tempEployee = employeeService.findById(employeeId);
		
		// throw exception if null
		if (tempEployee == null)
			throw new RuntimeException("Employee id not found - " + employeeId);
		
		employeeService.daleteById(employeeId);
		
		return "Deleted emploee id - " + employeeId;
	}
}
```
