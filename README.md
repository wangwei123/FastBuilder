FastBuilder是一个快速开发以及高性能，高扩展性的ORM框架，灵活支持多数据库切换，读写分离，同时支持Mysql和Oracle数据库, 并且上手快，在DAO层开发效率节约50%以上, 欢迎加入FastBuilder技术交流群：236719790

**FastBuilder目前支持的功能如下：**
* 1. CURD, Oracle和Mysql自动分页查询功能
* 2. 通过注解配置事务
* 3. 通过注解配置数据库的读写分离
* 4. Model层采用约定高于配置的方式，极大简化开发流程
* 5. 性能测试，根据网友对其它持久层框架(SpringJDBC,Mybatis,Hibernate)的测试，1万条数据查询前1000条数据(网友测试)，并循环执行求平均时间。
* 同样的测试方式，FastBuilder使用20万条数据查询前1000条数据(20倍的数据量)，并循环执行求平均时间，FastBuilder的性能略高于其它框架
* 6. **代码生成工具完成，一键生成Model, Service, Controller层代码**

**后续开发计划：**
* 1. 继续优化性能
* 2. 继续追求最简开发模式
* 3. 增加批量新增，批量修改，批量删除等功能(目前不支持，只能循环单条插入等，性能不如批量快)
* 4. 将开源Demo中的系统管理功能完善，包含用户管理，角色管理，权限管理等基础功能。


**快速开始：**
* 第一步：在项目中加入FastBuilder源码：
* * 几个包文件，直接copy进项目中即可

* 第二步：在Spring配置文件中配置数据源：

```xml
<!--主数据库，读写-->
<bean id="masterDataSource" class="com.jolbox.bonecp.BoneCPDataSource"  
    destroy-method="close">  
    <property name="driverClass" value="${db.driver}" />  
    <property name="jdbcUrl" value="${db.master.url}" />  
    <property name="username" value="${db.master.username}" />  
    <property name="password" value="${db.master.password}" />  
</bean>

<!--从数据库，只读-->
<bean id="slaveDataSource" class="com.jolbox.bonecp.BoneCPDataSource"  
        destroy-method="close">  
    <property name="driverClass" value="${db.driver}" />  
    <property name="jdbcUrl" value="${db.slave.url}" />  
    <property name="username" value="${db.slave.username}" />  
    <property name="password" value="${db.slave.password}" />  
</bean>

<!--事务管理-->
<bean id="transactionInterceptor" class="com.ch.fastbuilder.Interceptor.TransactionInterceptor"></bean>
<aop:config>  
    <aop:pointcut id="transactionPoint" expression="execution(* com..service..*.*(..))" />  
    <aop:aspect ref="transactionInterceptor">  
        <aop:before method="before" pointcut-ref="transactionPoint" />  
    </aop:aspect>  
    <aop:aspect ref="transactionInterceptor">  
        <aop:after-returning method="afterReturning" pointcut-ref="transactionPoint" />  
    </aop:aspect> 
    <aop:aspect ref="transactionInterceptor">  
        <aop:after-throwing method="afterThrowing" pointcut-ref="transactionPoint" />  
    </aop:aspect> 
</aop:config>  
    
```

* 上面两步后，可以进入开发了，本框架轻度依赖Spring框架，所以项目中必须引入spring框架：


**模型类创建：**
* 只需要继承Model父类，在构造函数中设置主键名，表名，字段名即可

```java

//=====================================================================================
//=======================Model Layer===================================================
//=====================================================================================

package com.ch.sys.model;

import com.ch.fastbuilder.model.Model;

public class Employee extends Model {
	
	public static String TABLE = "employee";
	public static String ID = "id";
	public static String ACCOUNT = "account";
	public static String PASSWORD = "password";
	public static String NICKNAME = "nickname";
	public static String GENDER = "gender";
	public static String HEAD_URL = "head_url";
	public static String TYPE = "type";
	public static String CREATED = "created";
	
	public Employee() {
		super.setPrimaryKey(ID);
		super.setTableName(TABLE);
		//super.setGenerationType(GenerationType.UUID);
		super.setGenerationType(GenerationType.IDENTITY);
		super.setColumns(ID,ACCOUNT,PASSWORD,NICKNAME,GENDER,HEAD_URL,TYPE,CREATED);
	}
	
	public static Employee getByAccountPwd(String account, String password) {
		Map<String, Object> params = new HashMap<String, Object>();
		params.put("account", account);
		params.put("password", SHA1.signature(password));
		
		String sql = "account=:account and password=:password";
		Employee employee = Model.Where(sql, params).get(Employee.class);
		
		return employee;
	}
}

//=====================================================================================
//=======================Service Layer=================================================
//=====================================================================================

@Service("employeeService")
public class EmployeeService {

    @Transaction(readOnly = true, DataSource = "masterDataSource")
	public Employee getByAccountPwd(String account, String password) {
		Employee employee = Employee.getByAccountPwd(account, password);
		return employee;
	}
}

//=====================================================================================
//=======================Controller Layer with Spring MVC==============================
//=====================================================================================

@Controller
@RequestMapping(Constants.REST_WEB_URL)
public class EmployeeController extends BaseController {
    
	@Autowired
	EmployeeService employeeService;
	
	@Autowired
	RoleService roleService;
	
	/**
	 * 员工登陆
	 * @return
	 */
	@ResponseBody
	@RequestMapping(value="/employee/login",method=RequestMethod.POST)
	public Response login(@RequestBody Employee model, HttpServletRequest request) {
		Response response = Response.newResponse();
		
		String account =  model.getString(Employee.ACCOUNT);
		String password = model.getString(Employee.PASSWORD);
		
		Employee employee = employeeService.getByAccountPwd(account, password);
		if(employee == null) {
			return response.ACCOUNT_PASS_ERROR();
		}
		
		Role role = roleService.getByEmployeeId(employee.getLong(Employee.ID));
		employee.set("role", role);
		SessionUtils.set(request, employee);
		
		return response.ok(employee);
	}
}

```


**关联查询：**

```java
public class Role extends Model {

    /**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	
	public static String TABLE = "role";
	public static String ID = "id";
	public static String NAME = "name";
	public static String CREATED = "created";
	
	public Role() {
		super.setPrimaryKey(ID);
		super.setTableName(TABLE);
		//super.setGenerationType(GenerationType.UUID);
		super.setGenerationType(GenerationType.IDENTITY);
		super.setColumns(ID, NAME, CREATED);
	}
	
	public static PageResult findByPage(ParamMap params) {
		BuilderModel builder = Model.InitParams(params);
		PageResult pageResult = builder.findPage(Role.class);
		
		return pageResult;
	}
	
	public static Role getByEmployeeId(Long employeeId) {
		Map<String, Object> params = new HashMap<String, Object>();
		params.put("employee_id", employeeId);
		
		BuilderModel builder = Model.InitParams(params);
		builder.select("r.id, r.name, r.created");
		builder.from("employee_role emr");
		builder.innerJoin("role r on emr.role_id = r.id");
		builder.where("emr.employee_id = :employee_id");
		
		Role role = builder.get(Role.class);
		return role;
	}
}
```


**新增一条数据：**
* 从客户端post过来的数据，Model从request中获取，然后转换为Role对象
* 任何一个继承Model的对象，可以直接调用create进行新增操作

```java

    //新增操作
    public void add() {
		Employee employee = new Employee();
        employee.set(Employee.ACCOUNT, "wangwei");
    	employee.set(Employee.PASSWORD, "111111");
		employee.create();
        
        //或者这样执行create方法
        Model.Create(employee);
	}
    
```

**修改一条数据：**
* 从客户端post过来的数据，Model从request中获取，然后转换为Role对象
* 任何一个继承Model的对象，可以直接调用update进行新增操作

```java

    //修改操作
    public void update() {
		Employee employee = new Employee();
        employee.set(Employee.ID, 1);
        employee.set(Employee.ACCOUNT, "LiMing");
        employee.set(Employee.PASSWORD, "123456");
		employee.update();
        
        //或者这样执行update方法
        Model.Update(employee);
	}
    
```

**删除一条数据：**
* 从客户端post过来的数据，Model从request中获取，然后转换为Role对象
* 任何一个继承Model的对象，可以直接调用delete进行新增操作

```java
    
    //删除操作
    public void delete() {
		Employee employee = new Employee();
        employee.set(Employee.ID, 1);
		employee.delete();
        
        //或者这样执行delete方法
        Model.Delete(employee);
        
        //或者这样执行delete方法
        //Model.Delete(id, Employee.class);
        Model.Delete(1, Employee.class);
	}
    
```

**读写分离，事务管理：**
```java

@Service("roleService")
public class RoleService {

    @Transaction(readOnly = true, DataSource = "slaveDataSource")
	public Response findPage(ParamMap param) {
		Response response = Response.newResponse();
		
		PageResult pageResult = Role.findByPage(param);
		return response.setPageResults(pageResult);
	}
	
	@Transaction(readOnly = true, DataSource = "slaveDataSource")
	public Role getByEmployeeId(Long employeeId) {
		Role role = Role.getByEmployeeId(employeeId);
		return role;
	}
	
	@Transaction(readOnly = false, DataSource = "masterDataSource")
	public Response add(Role role) {
		Response response = Response.newResponse();
		
		Timestamp time = new Timestamp(System.currentTimeMillis());
		role.set(Role.CREATED, time);
		role.create();
		
		return response.OK();
	}
	
	@Transaction(readOnly = false, DataSource = "masterDataSource")
	public Response update(Role role) {
		Response response = Response.newResponse();
		role.update();
		
		return response.OK();
	}
}

```

**单条数据查询：**

```java
package com.ch.sys.service;

import java.util.HashMap;
import java.util.Map;

import org.springframework.stereotype.Service;

import com.ch.fastbuilder.model.Model;
import com.ch.sys.model.Employee;
import com.ch.sys.utils.SHA1;

@Service("employeeService")
public class EmployeeService {

    public Employee getByAccountPwd(String account, String password) {
		Map<String, Object> params = new HashMap<String, Object>();
		params.put("account", account);
		params.put("password", SHA1.signature(password));
		
		String sql = "account=:account and password=:password";
		Employee employee = Model.Where(sql, params).get(Employee.class);
		
		return employee;
	}
}
```

**分页查询：**
* Model.InitParams(request)，只需要客户端传入pageIndex和pageSize，可以自动分页

```java

public static PageResult findByPage(ParamMap params) {
	BuilderModel builder = Model.InitParams(params);
	PageResult pageResult = builder.findPage(Role.class);
	
	return pageResult;
}

* 或者这样分页查询
List<Role> roles = Model.Limit(0, 10).list(Role.class);

* 或者这样分页查询
Map<String,Object> params = new HashMap<String,Object>();
params.put("id", 5);
params.put("date", new Date());
List<Role> roles = Model.Where("role_id > :id and created < :date", params).limit(1, 10).list(Role.class);

* 这么写太长了么？可以这么写
BuilderModel builder = Model.Where("role_id > :id and created < :date", params);
List<Role> roles = builder.limit(1, 10).list(Role.class);

* 可以自定义查询字段
List<Role> roles = Model.Select("id, name, created").limit(1, 10).list(Role.class);


```

**执行SQL语句：**
* 可以执行任何SQL语句

```java

Map<String,Object> params = new HashMap<String,Object>();
params.put("name", "修改名称");

* 修改操作
Model.SQL("update role set name = :name").withParams(params).excute();

* 查询操作
Model.SQL("select * from Role where id > :id").withParams(params).list(Role.class);

* 查询数量，更多操作这里不列出来了，具体看demo

```
