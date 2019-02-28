# 中文翻译版（Chinese Translate Version）
MysqliDb - 带有预处理语句的简单MySQLi包装器和对象映射器

<hr>

### 目录

**[安装](#安装)**
**[初始化](#初始化)**
**[对象映射](#对象映射)**
---
**[insert-插入查询](#insert-插入查询)**
**[update-更新查询](#update-更新查询)**
**[select-选择查询](#select-选择查询)**  
**[delete-删除查询](#delete-删除查询)**  
**[insert-插入数据](#insert-插入数据)** 
---
**[插入XML](#插入XML)**  
**[分页](#分页)**  
**[结果转换/地图](#结果转换/地图)**  
**[定义返回类型](#定义返回类型)**  
---
**[运行原生SQL查询](#运行原生SQL查询)**  
**[where-条件查询](#where-条件查询)**  
**[查询关键字](#查询关键字)** 
**[子查询](#子查询)** 
---
**[排序方法](#排序方法)**  
**[分组方法](#分组方法)**  
**[JOIN方法](#JOIN方法)**  
**[属性共享](#属性共享)**  
**[Join-连接表](#join-连接表)**  
**[Subqueries-子查询](#subqueries-子查询)**  
**[EXISTS / NOT EXISTS 条件](#exists--not-exists-condition)**  
---
**[Has方法](#Has方法)**  
**[助手方法](#助手方法)**  
**[Transaction Helpers](#transaction-helpers)**  
**[错误助手](#错误助手)**  
---
**[查询执行时间基准测试](#查询执行时间基准测试)**  
**[表锁定](#表锁定)**  

## Support Me

This software is developed during my free time and I will be glad if somebody will support me.

Everyone's time should be valuable, so please consider donating.

[Donate with paypal](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=a%2ebutenka%40gmail%2ecom&lc=DO&item_name=mysqlidb&currency_code=USD&bn=PP%2dDonationsBF%3abtn_donateCC_LG%2egif%3aNonHosted)

### 安装
要使用此类，首先将 `MysqliDb.php` 导入到您的项目中，并 `require` 它。

```php
require_once ('MysqliDb.php');
```

### 使用 *composer* 安装
可以通过 `composer` 安装
```
composer require thingengineer/mysqli-database-class:dev-master
```

### 初始化
默认情况下使用 utf8 charset 进行简单初始化：
```php
$db = new MysqliDb ('host', 'username', 'password', 'databaseName');
```

高级初始化：
```php
$db = new MysqliDb (Array (
                'host' => 'host',
                'username' => 'username', 
                'password' => 'password',
                'db'=> 'databaseName',
                'port' => 3306,
                'prefix' => 'my_',
                'charset' => 'utf8'));
```
表前缀，端口和数据库字符集参数是可选的。如果没有 charset 应设置 charset，请将其设置为 null

还可以重用已连接的 mysqli 对象：

如果在对象创建期间未设置表前缀，则可以稍后通过单独调用来设置它：
```php
$mysqli = new mysqli ('host', 'username', 'password', 'databaseName');
$db = new MysqliDb ($mysqli);
```

如果在对象创建期间未设置 *表前缀*，则可以稍后通过单独调用来设置它：
```php
$db->setPrefix ('my_');
```

如果将删除与 mysql 的连接(也就是断开了连接)，`Mysqlidb` 将尝试自动重新连接到数据库一次。要禁用此行为使用请：
```php
$db->autoReconnect = false;
```

如果你需要从另一个类或函数中获取已经创建的 mysqliDb 对象
```php
    function init () {
        // db在这里保持私有
        $db = new MysqliDb ('host', 'username', 'password', 'databaseName');
    }
    ...
    function myfunc () {
        // 获取在 init() 中创建的db对象
        $db = MysqliDb::getInstance();
        ...
    }
```

### 多个数据库连接
如果需要连接到多个数据库，请使用以下方法：
```php
$db->addConnection('slave', Array (
                'host' => 'host',
                'username' => 'username',
                'password' => 'password',
                'db'=> 'databaseName',
                'port' => 3306,
                'prefix' => 'my_',
                'charset' => 'utf8')
);
```
要选择数据库，请使用 `connection()` 方法
```php
$users = $db->connection('slave')->get('users');
```

### 对象映射
`dbObject.php` 是一个构建在 *mysqliDb* 之上的对象映射库，用于提供模型表示功能。有关更多信息，请参见 <a href='dbObject.md'>dbObject 手册</a>

### insert-插入查询
简单的例子：
```php
$data = Array ("login" => "admin",
               "firstName" => "John",
               "lastName" => 'Doe'
);
$id = $db->insert ('users', $data);
if($id)
    echo 'user was created. Id=' . $id;
```

插入功能使用（Insert with functions use）：
```php
$data = Array (
	'login' => 'admin',
    'active' => true,
	'firstName' => 'John',
	'lastName' => 'Doe',
	'password' => $db->func('SHA1(?)',Array ("secretpassword+salt")),
	// password = SHA1('secretpassword+salt')
	'createdAt' => $db->now(),
	// createdAt = NOW()
	'expires' => $db->now('+1Y')
	// expires = NOW() + interval 1 year
	// Supported intervals [s]econd, [m]inute, [h]hour, [d]day, [M]onth, [Y]ear
);

$id = $db->insert ('users', $data);
if ($id)
    echo 'user was created. Id=' . $id;
else
    echo 'insert failed: ' . $db->getLastError();
```

插入重复键更新（Insert with on duplicate key update）：
```php
$data = Array ("login" => "admin",
               "firstName" => "John",
               "lastName" => 'Doe',
               "createdAt" => $db->now(),
               "updatedAt" => $db->now(),
);
$updateColumns = Array ("updatedAt");
$lastInsertId = "id";
$db->onDuplicate($updateColumns, $lastInsertId);
$id = $db->insert ('users', $data);
```

一次插入多个数据集：
```php
$data = Array(
    Array ("login" => "admin",
        "firstName" => "John",
        "lastName" => 'Doe'
    ),
    Array ("login" => "other",
        "firstName" => "Another",
        "lastName" => 'User',
        "password" => "very_cool_hash"
    )
);
$ids = $db->insertMulti('users', $data);
if(!$ids) {
    echo 'insert failed: ' . $db->getLastError();
} else {
    echo 'new users inserted with following id\'s: ' . implode(', ', $ids);
}
```

如果所有数据集只有相同的键，则可以简化：
```php
$data = Array(
    Array ("admin", "John", "Doe"),
    Array ("other", "Another", "User")
);
$keys = Array("login", "firstName", "lastName");

$ids = $db->insertMulti('users', $data, $keys);
if(!$ids) {
    echo 'insert failed: ' . $db->getLastError();
} else {
    echo 'new users inserted with following id\'s: ' . implode(', ', $ids);
}
```

### replace-替换查询
<a href='https://dev.mysql.com/doc/refman/5.0/en/replace.html'>Replace()</a>方法实现与 `insert()` 是相同的 API;

### update-更新查询
```php
$data = Array (
	'firstName' => 'Bobby',
	'lastName' => 'Tables',
	'editCount' => $db->inc(2),
	// editCount = editCount + 2;
	'active' => $db->not()
	// active = !active;
);
$db->where ('id', 1);
if ($db->update ('users', $data))
    echo $db->count . ' records were updated';
else
    echo 'update failed: ' . $db->getLastError();
```

`update()` 还支持限制参数：
```php
$db->update ('users', $data, 10);
// Gives: UPDATE users SET ... LIMIT 10
```

### select-选择查询
在任何 select/get 函数调用之后，返回的行都存储在 *$count* 变量中：
```php
$users = $db->get('users'); //contains an Array of all users 
$users = $db->get('users', 10); //contains an Array 10 users
```

或选择自定义列设置。也可以使用函数：

```php
$cols = Array ("id", "name", "email");
$users = $db->get ("users", null, $cols);
if ($db->count > 0)
    foreach ($users as $user) { 
        print_r ($user);
    }
```

或者只选择一行：

```php
$db->where ("id", 1);
$user = $db->getOne ("users");
echo $user['id'];

$stats = $db->getOne ("users", "sum(id), count(*) as cnt");
echo "total ".$stats['cnt']. "users found";
```

或选择一个列值或功能结果：

```php
$count = $db->getValue ("users", "count(*)");
echo "{$count} users found";
```

从多行中选择一个列值或函数结果：
```php
$logins = $db->getValue ("users", "login", null);
// select login from users
$logins = $db->getValue ("users", "login", 5);
// select login from users limit 5
foreach ($logins as $login)
    echo $login;
```

您还可以将 *.CSV* 或 *.XML* 数据加载到特定表中。
要插入.csv数据，请使用以下语法：
```php
$path_to_file = "/home/john/file.csv";
$db->loadData("users", $path_to_file);
```
这将在文件夹 **/home/john/**（john 的主目录）中加载名为 **file.csv** 的 *.csv* 文件。
您还可以附加可选的选项数组。
有效选项包括：

```php
Array(
	"fieldChar" => ';', 	// 分隔数据的字符
	"lineChar" => '\r\n', 	// 分隔行的字符
	"linesToIgnore" => 1	// 要忽略的行数在进口开始时
);
```

使用附加它们（Attach them using）：
```php
$options = Array("fieldChar" => ';', "lineChar" => '\r\n', "linesToIgnore" => 1);
$db->loadData("users", "/home/john/file.csv", $options);
// LOAD DATA ...
```

您可以指定使用 **LOCAL DAT** A而不是 **DATA**：
```php
$options = Array("fieldChar" => ';', "lineChar" => '\r\n', "linesToIgnore" => 1, "loadDataLocal" => true);
$db->loadData("users", "/home/john/file.csv", $options);
// LOAD DATA LOCAL ...
```

### 插入XML
要将 XML 数据加载到表中，可以使用方法 **loadXML**。
语法对于 loadData 语法来说是微不足道的。
```php
$path_to_file = "/home/john/file.xml";
$db->loadXML("users", $path_to_file);
```

您还可以添加可选参数。
有效参数：
```php
Array(
	"linesToIgnore" => 0,		// Amount of lines / rows to ignore at the beginning of the import
	"rowTag"	=> "<user>"	// The tag which marks the beginning of an entry
)
```

用法：
```php
$options = Array("linesToIgnore" => 0, "rowTag"	=> "<user>"):
$path_to_file = "/home/john/file.xml";
$db->loadXML("users", $path_to_file, $options);
```

### 分页
使用 paginate() 而不是 get() 来获取分页结果
```php
$page = 1;
// 将页面限制设置为每页 2 个结果。默认是 20
$db->pageLimit = 2;
$products = $db->arraybuilder()->paginate("products", $page);
echo "showing $page out of " . $db->totalPages;

```

### 结果转换/地图
而不是获得纯粹的结果数组，可以得到带有所需键的关联数组。
如果在 get() 中只设置了2个要获取的字段，则在其余情况下，方法将返回数组 array($k => $v) 和数组 ($k => array ($v, $v)) 中的结果。

```php
$user = $db->map ('login')->ObjectBuilder()->getOne ('users', 'login, id');
Array
(
    [user1] => 1
)

$user = $db->map ('login')->ObjectBuilder()->getOne ('users', 'id,login,createdAt');
Array
(
    [user1] => stdClass Object
        (
            [id] => 1
            [login] => user1
            [createdAt] => 2015-10-22 22:27:53
        )

)
```

### 定义返回类型
MysqliDb 可以 以3种不同的格式返回结果：数组数组，对象数组和 Json 字符串。
要选择返回类型，请使用 ArrayBuilder()，ObjectBuilder() 和 JsonBuilder() 方法。
请注意，ArrayBuilder() 是默认的返回类型
```php
// Array return type
$= $db->getOne("users");
echo $u['login'];
// Object return type
$u = $db->ObjectBuilder()->getOne("users");
echo $u->login;
// Json return type
$json = $db->JsonBuilder()->getOne("users");
```

### 运行原始SQL查询
```php
$users = $db->rawQuery('SELECT * from users where id >= ?', Array (10));
foreach ($users as $user) {
    print_r ($user);
}
```
为了避免长时间检查有几个辅助函数来处理原始查询选择结果：

获得 1行 结果：
```php
$user = $db->rawQueryOne ('select * from users where id=?', Array(10));
echo $user['login'];
// Object return type
$user = $db->ObjectBuilder()->rawQueryOne ('select * from users where id=?', Array(10));
echo $user->login;
```
获取 1列 值作为字符串：
```php
$password = $db->rawQueryValue ('select password from users where id=? limit 1', Array(10));
echo "Password is {$password}";
NOTE: for a rawQueryValue() to return string instead of an array 'limit 1' should be added to the end of the query.
```
从多行获取 1列 值：
```php
$logins = $db->rawQueryValue ('select login from users limit 10');
foreach ($logins as $login)
    echo $login;
```

更高级的例子：
```php
$params = Array(1, 'admin');
$users = $db->rawQuery("SELECT id, firstName, lastName FROM users WHERE id = ? AND login = ?", $params);
print_r($users); // 包含返回行的数组

// 将处理任何SQL查询
$params = Array(10, 1, 10, 11, 2, 10);
$q = "(
    SELECT a FROM t1
        WHERE a = ? AND B = ?
        ORDER BY a LIMIT ?
) UNION (
    SELECT a FROM t2 
        WHERE a = ? AND B = ?
        ORDER BY a LIMIT ?
)";
$resutls = $db->rawQuery ($q, $params);
print_r ($results); // 包含返回行的数组
```

### where条件查询
`where()`, `orWhere()`, `having()` and `orHaving()` 方法允许你指定和查询具有的条件。where() 支持的所有条件也由 having() 支持。

警告：为了仅使用列到列的比较，应将条件用作列名或函数不能作为绑定变量传递

带变量的 Regular == 运算符：
```php
$db->where ('id', 1);
$db->where ('login', 'admin');
$results = $db->get ('users');
// Gives: SELECT * FROM users WHERE id=1 AND login='admin';
```

```php
$db->where ('id', 1);
$db->having ('login', 'admin');
$results = $db->get ('users');
// Gives: SELECT * FROM users WHERE id=1 HAVING login='admin';
```


使用列到列比较的常规 == 运算符：
```php
// WRONG
$db->where ('lastLogin', 'createdAt');
// CORRECT
$db->where ('lastLogin = createdAt');
$results = $db->get ('users');
// Gives: SELECT * FROM users WHERE lastLogin = createdAt;
```

```php
$db->where ('id', 50, ">=");
// or $db->where ('id', Array ('>=' => 50));
$results = $db->get ('users');
// Gives: SELECT * FROM users WHERE id >= 50;
```

BETWEEN / NOT BETWEEN:
```php
$db->where('id', Array (4, 20), 'BETWEEN');
// or $db->where ('id', Array ('BETWEEN' => Array(4, 20)));

$results = $db->get('users');
// Gives: SELECT * FROM users WHERE id BETWEEN 4 AND 20
```

IN / NOT IN:
```php
$db->where('id', Array(1, 5, 27, -1, 'd'), 'IN');
// or $db->where('id', Array( 'IN' => Array(1, 5, 27, -1, 'd') ) );

$results = $db->get('users');
// Gives: SELECT * FROM users WHERE id IN (1, 5, 27, -1, 'd');
```

OR CASE:
```php
$db->where ('firstName', 'John');
$db->orWhere ('firstName', 'Peter');
$results = $db->get ('users');
// Gives: SELECT * FROM users WHERE firstName='John' OR firstName='peter'
```

NULL 比较:
```php
$db->where ("lastName", NULL, 'IS NOT');
$results = $db->get("users");
// Gives: SELECT * FROM users where lastName IS NOT NULL
```

LIKE 比较:
```php
$db->where ("fullName", 'John%', 'like');
$results = $db->get("users");
// Gives: SELECT * FROM users where fullName like 'John%'
```

你也可以在以下条件下使用 raw：
```php
$db->where ("id != companyId");
$db->where ("DATE(createdAt) = DATE(lastLogin)");
$results = $db->get("users");
```

或者带变量的原始条件：
```php
$db->where ("(id = ? or id = ?)", Array(6,2));
$db->where ("login","mike")
$res = $db->get ("users");
// Gives: SELECT * FROM users WHERE (id = 6 or id = 2) and login='mike';
```


找到匹配的总行数。简单的分页示例：
```php
$offset = 10;
$count = 15;
$users = $db->withTotalCount()->get('users', Array ($offset, $count));
echo "Showing {$count} from {$db->totalCount}";
```

### 查询关键字
添加 LOW PRIORITY | DELAYED | HIGH PRIORITY | IGNORE 和其余的 mysql 关键字为 INSERT (), REPLACE (), GET (), UPDATE (), DELETE() 方法或 FOR UPDATE | 将共享模式锁定到 SELECT()：
```php
$db->setQueryOption ('LOW_PRIORITY')->insert ($table, $param);
// GIVES: INSERT LOW_PRIORITY INTO table ...
```
```php
$db->setQueryOption ('FOR UPDATE')->get ('users');
// GIVES: SELECT * FROM USERS FOR UPDATE;
```

您还可以使用一系列关键字：
```php
$db->setQueryOption (Array('LOW_PRIORITY', 'IGNORE'))->insert ($table,$param);
// GIVES: INSERT LOW_PRIORITY IGNORE INTO table ...
```

同样，关键字也可以在SELECT查询中使用：
```php
$db->setQueryOption ('SQL_NO_CACHE');
$db->get("users");
// GIVES: SELECT SQL_NO_CACHE * FROM USERS;
```

（可选）您可以使用方法链接多次调用，而无需反复引用对象：

```php
$results = $db
	->where('id', 1)
	->where('login', 'admin')
	->get('users');
```

### delete-删除查询
```php
$db->where('id', 1);
if($db->delete('users')) echo 'successfully deleted';
```


### 排序方法
```php
$db->orderBy("id","asc");
$db->orderBy("login","Desc");
$db->orderBy("RAND ()");
$results = $db->get('users');
// Gives: SELECT * FROM users ORDER BY id ASC,login DESC, RAND ();
```

按值排序示例：
```php
$db->orderBy('userGroup', 'ASC', array('superuser', 'admin', 'users'));
$db->get('users');
// Gives: SELECT * FROM users ORDER BY FIELD (userGroup, 'superuser', 'admin', 'users') ASC;
```

如果您正在使用 setPrefix() 功能并且需要在 orderBy() 方法中使用表名，请确保使用 `` 转义表名。
If you are using setPrefix () functionality and need to use table names in orderBy() method make sure that table names are escaped with ``.

```php
$db->setPrefix ("t_");
$db->orderBy ("users.id","asc");
$results = $db->get ('users');
// WRONG: That will give: SELECT * FROM t_users ORDER BY users.id ASC;

$db->setPrefix ("t_");
$db->orderBy ("`users`.id", "asc");
$results = $db->get ('users');
// CORRECT: That will give: SELECT * FROM t_users ORDER BY t_users.id ASC;
```

### 分组方法
```php
$db->groupBy ("name");
$results = $db->get ('users');
// Gives: SELECT * FROM users GROUP BY name;
```

通过 tenantID 与表用户一起使用 LEFT JOIN 加入表产品
### JOIN方法
```php
$db->join("users u", "p.tenantID=u.tenantID", "LEFT");
$db->where("u.id", 6);
$products = $db->get ("products p", null, "u.name, p.productName");
print_r ($products);
```

### 加入条件
添加 AND 条件以加入语句：
```php
$db->join("users u", "p.tenantID=u.tenantID", "LEFT");
$db->joinWhere("users u", "u.tenantID", 5);
$products = $db->get ("products p", null, "u.name, p.productName");
print_r ($products);
// Gives: SELECT  u.login, p.productName FROM products p LEFT JOIN users u ON (p.tenantID=u.tenantID AND u.tenantID = 5)
```
添加 OR 条件以加入语句：
```php
$db->join("users u", "p.tenantID=u.tenantID", "LEFT");
$db->joinOrWhere("users u", "u.tenantID", 5);
$products = $db->get ("products p", null, "u.name, p.productName");
print_r ($products);
// Gives: SELECT  u.login, p.productName FROM products p LEFT JOIN users u ON (p.tenantID=u.tenantID OR u.tenantID = 5)
```

### 属性共享
也可以复制属性

```php
$db->where ("agentId", 10);
$db->where ("active", true);

$customers = $db->copy ();
$res = $customers->get ("customers", Array (10, 10));
// SELECT * FROM customers where agentId = 10 and active = 1 limit 10, 10

$cnt = $db->getValue ("customers", "count(id)");
echo "total records found: " . $cnt;
// SELECT count(id) FROM users where agentId = 10 and active = 1
```

### 子查询
子查询初始化

没有别名的子查询 init 用于 insert/updates/where Eg：（select * from users）
```php
$sq = $db->subQuery();
$sq->get ("users");
```

指定了在 JOIN 中使用的别名的子查询。例如：(select * from users) sq
```php
$sq = $db->subQuery("sq");
$sq->get ("users");
```

子查询选择（Subquery in selects）：
```php
$ids = $db->subQuery ();
$ids->where ("qty", 2, ">");
$ids->get ("products", null, "userId");

$db->where ("id", $ids, 'in');
$res = $db->get ("users");
// Gives SELECT * FROM users WHERE id IN (SELECT userId FROM products WHERE qty > 2)
```

插入中的子查询（Subquery in inserts）：
```php
$userIdQ = $db->subQuery ();
$userIdQ->where ("id", 6);
$userIdQ->getOne ("users", "name"),

$data = Array (
    "productName" => "test product",
    "userId" => $userIdQ,
    "lastUpdated" => $db->now()
);
$id = $db->insert ("products", $data);
// Gives INSERT INTO PRODUCTS (productName, userId, lastUpdated) values ("test product", (SELECT name FROM users WHERE id = 6), NOW());
```

连接中的子查询：
```php
$usersQ = $db->subQuery ("u");
$usersQ->where ("active", 1);
$usersQ->get ("users");

$db->join($usersQ, "p.userId=u.id", "LEFT");
$products = $db->get ("products p", null, "u.login, p.productName");
print_r ($products);
// SELECT u.login, p.productName FROM products p LEFT JOIN (SELECT * FROM t_users WHERE active = 1) u on p.userId=u.id;
```

### EXISTS / NOT EXISTS condition
```php
$sub = $db->subQuery();
    $sub->where("company", 'testCompany');
    $sub->get ("users", null, 'userId');
$db->where (null, $sub, 'exists');
$products = $db->get ("products");
// Gives SELECT * FROM products WHERE EXISTS (select userId from users where company='testCompany')
```

### Has方法
一个方便的函数，如果存在至少一个满足where条件的元素，则返回TRUE，该元素在此之前调用“where”方法。
```php
$db->where("user", $user);
$db->where("password", md5($password));
if($db->has("users")) {
    return "You are logged";
} else {
    return "Wrong user/password";
}
``` 
### 助手方法
断开与数据库的连接：
```php
    $db->disconnect();
```

重新连接，以防 mysql 连接死机：
```php
if (!$db->ping())
    $db->connect()
```

获取上次执行的 SQL查询：
请注意，函数返回 SQL查询 仅用于调试目的，因为它的执行很可能会因 char 变量缺少引号而失败。
```php
    $db->get('users');
    echo "Last executed query was ". $db->getLastQuery();
```

检查表(table)是否存在：
```php
    if ($db->tableExists ('users'))
        echo "hooray";
```

mysqli_real_escape_string() wrapper:
```php
    $escaped = $db->escape ("' and 1=1");
```

### Transaction helpers
请记住，事务正在处理 innoDB 表。
如果插入失败，则回滚事务：
```php
$db->startTransaction();
...
if (!$db->insert ('myTable', $insertData)) {
    //Error while saving, cancel new record
    $db->rollback();
} else {
    //OK
    $db->commit();
}
```


### 错误助手
执行查询后，您可以选择检查是否存在错误。您可以获取MySQL错误字符串或上次执行的查询的错误代码。
```php
$db->where('login', 'admin')->update('users', ['firstName' => 'Jack']);

if ($db->getLastErrno() === 0)
    echo 'Update succesfull';
else
    echo 'Update failed. Error: '. $db->getLastError();
```

### 查询执行时间基准测试
要跟踪查询执行时间，应调用 setTrace() 函数。
```php
$db->setTrace (true);
// As a second parameter it is possible to define prefix of the path which should be striped from filename
// $db->setTrace (true, $_SERVER['SERVER_ROOT']);
$db->get("users");
$db->get("test");
print_r ($db->trace);
```

```
    [0] => Array
        (
            [0] => SELECT  * FROM t_users ORDER BY `id` ASC
            [1] => 0.0010669231414795
            [2] => MysqliDb->get() >>  file "/avb/work/PHP-MySQLi-Database-Class/tests.php" line #151
        )

    [1] => Array
        (
            [0] => SELECT  * FROM t_test
            [1] => 0.00069189071655273
            [2] => MysqliDb->get() >>  file "/avb/work/PHP-MySQLi-Database-Class/tests.php" line #152
        )

```

### 表锁定
要锁定表，可以将 **lock** 方法与 **setLockMethod** 一起使用。
以下示例将锁定表 **用户** 以进行 **写入** 访问。
```php
$db->setLockMethod("WRITE")->lock("users");
```
调用另一个 **->lock()** 将删除第一个锁。
你也可以使用
```php
$db->unlock();
```
解锁以前锁定的表格。
要锁定多个表，可以使用数组。
例子：
```php
$db->setLockMethod("READ")->lock(array("users", "log"));
```
这将锁定表 **users** 和 **log** 仅用于 **READ** 访问。
确保之后使用 **unlock()**，否则您的 **table** 将保持锁定状态！


