# jooq-cookbook
## 使用JOOQ高效编写Java数据库应用

> If your pictures aren’t good enough, you aren’t close enough. 

> – Robert Capa

围绕着关系代数构建的数据库系统是软件开发的核心之一，对很多软件工程来说，数据库建模就是数据模型的核心。

为了使编程语言，尤其是面向对象的语言更好和关系数据模型进行配合，出现了大量的ORM框架。比较成熟的模式是在设计了数据库的schema之后，还需要构建相关的模型和DAO（Data Access Object）代码，来实现数据库模型和对象之间的映射（OR-Mapping）。

## why JOOQ

相比较Hibernate、Mybatis等大名鼎鼎的ORM框架来说，JOOQ只能算是一个小字辈。但它之所以能够脱颖而出，是因为它找到了正确的设计思想。

**数据库模型是数据模型的核心**。很多ORM框架的目标是要扭转核心为对象，试图通过对象来生成数据库Schema，如Ebean、JPA等，但这是有问题的：首先相比对象，数据存储模型是很难变化，而且变化起来迁移难度很大；其次相比面向对象的模型，关系模型是面向数据的，具有形式化的基础【创建者Codd因此获得图领奖】，更加简洁、一致。关系模型的成果之一就是结构化查询语言SQL，现代数据库都针对SQL进行了大量的优化，越能控制SQL就越能把控数据库查询的性能。

相比其它ORM，JOOQ更注重于对[数据库模型的建模](http://stackoverflow.com/a/4208156/851099)，它是：

* 面向SQL
* 构建类型安全的SQL
* 强大的代码生成工具
* 方便的乐观锁等高级特性

可以看看官网是怎么[自卖自夸](http://www.jooq.org/doc/3.8/manual-single-page/#preface)的。如果说它有什么缺点，那就是他对一些商用数据库，如Oracle、SQL Server是收费的。

## JOOQ入门简介

让我们通过官网的[《7步了解JOOQ》](http://www.jooq.org/doc/3.8/manual-single-page/#jooq-in-7-steps)中的例子来认识一下JOOQ。

首先需要在JOOQ的[下载页面](http://www.jooq.org/download)下载它。如果使用的是开源版本，可以直接通过Maven等工具下载。

然后构建我们的数据库：

```
CREATE DATABASE `Library`;

USE `Library`;

CREATE TABLE `Author` (
  `id` int NOT NULL,
  `first_name` varchar(255) DEFAULT NULL,
  `last_name` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
);
```

然后可以按照[官网的方法](http://www.jooq.org/doc/3.8/manual-single-page/#jooq-in-7-steps-step3)生成代码。然后就可以在自己的代码中使用这些生成的代码了：

```
import static test.generated.Tables.*;
import static org.jooq.impl.DSL.*;

import java.sql.*;

import org.jooq.*;
import org.jooq.impl.*;

public class Main {
  public static void main(String[] args) {
    String userName = "root";
    String password = "";
    String url = "jdbc:mysql://localhost:3306/Library";
    // Connection is the only JDBC resource that we need
    // PreparedStatement and ResultSet are handled by jOOQ, internally
    try (Connection conn = DriverManager.getConnection(url, userName, password)) {
      DSLContext dsl = DSL.using(conn, SQLDialect.MYSQL);
      Result<Record> result = dsl.select().from(AUTHOR).fetch();
      for (Record r : result) {
        Integer id = r.getValue(AUTHOR.ID);
        String firstName = r.getValue(AUTHOR.FIRSTNAME);
        String lastName = r.getValue(AUTHOR.LASTNAME);
        System.out.println("ID: " + id + " first name: " + firstName + " last name: " + lastName);
      }
    } catch (Exception e) {
      e.printStackTrace();
    }
  }
}
```

so far, so good. 

JOOQ可以编写非常接近SQL的代码，非常直观和灵活，看到JOOQ代码你就知道对应的SQL是什么【what you see is what you get】，例如：

```
-- SQL --
// Select authors with books that are sold out
SELECT * FROM T_AUTHOR a
  WHERE EXISTS (SELECT 1 FROM T_BOOK
        WHERE T_BOOK.STATUS = 'SOLD OUT'
 AND T_BOOK.AUTHOR_ID = a.ID);

// Java
TAuthor a = T_AUTHOR.as("a");
create.selectFrom(a)
  .whereExists(create.selectOne().from(T_BOOK)
   .where(T_BOOK.STATUS.equal(TBookStatus.SOLD_OUT)
   .and(T_BOOK.AUTHOR_ID.equal(a.ID))))));
```

可以参考JOOQ的官方文档——[SQL Building](http://www.jooq.org/doc/3.8/manual-single-page/#sql-building)、[SQL execution](http://www.jooq.org/doc/3.8/manual-single-page/#sql-execution)

下面将重点介绍JOOQ在实际使用中可以提高编程效率的方面，和一些文档中没有提及的实际问题的解决方案。

## 代码生成

有很多建模工具能让我们很快速清晰地构建出数据库模型，但是构建好数据库模型之后，往往还需要编写对应的Java代码。JOOQ提供了很强大的[代码生成能力](http://www.jooq.org/doc/3.8/manual-single-page/#code-generation)，使我们不再需要手工构建映射。

如果对JOOQ代码生成能力还不熟悉，可以在回顾一下[JOOQ入门](http://www.jooq.org/doc/3.8/manual-single-page/#jooq-in-7-steps-step3)。

下面就看看怎样能用好JOOQ的代码生成能力吧。

### Tip 1 简化引入

JOOQ会在生成代码的根目录生成`Tables.java`，`Keys.java`等，可以直接`import`这些类实现库表的快速引入：

```
// Static imports for all global artefacts (if they exist)
import static test.generated.Keys.*;
import static test.generated.Routines.*;
import static test.generated.Sequences.*;
import static test.generated.Tables.*;
```

### Tip 2 使用带类型的Record

JOOQ生成的`AuthorRecord`，自带增删改查（CRUD）功能。任何一个**带主键**的表都能够生成这样的具有更新功能的Record对象：

```
// create
AuthorRecord author = dsl.newRecord(AUTHOR);
author.setFirstName("George");
author.setLastName("Orwell");
author.store();

// update
author = dsl.selectFrom(AUTHOR).where(AUTHOR.ID.eq(1)).fetchOne();
author.setFirstName("Changed");
author.store();

// read
author = dsl.newRecord(AUTHOR);
author.setId(3);
author.refresh();

// delete
author.delete();
```

可以看到，字段名称自动根据下划线 —— '_'为分隔符，生成了对应的Camel名字段，如`first_name`变为`firstName`。

### Tip 3 生成POJO

有了Schema，POJO也可以自动生成！

首先在生成配置中添加生成POJO的选项：

```
	<generate>
		<!-- Generate POJOs in addition to Record classes for usage of 
		    the ResultQuery.fetchInto(Class) API 
			Defaults to false -->
		<pojos>true</pojos>
	</generate>
```

重新运行生成工具，一个`Author`类就生成了：

```
public class Author implements Serializable {
    private Integer id;
    private String  firstName;
    private String  lastName;
    public Author() {}
    public Author(Author value) { ... }
    public Author(Integer id, String firstName, String lastName) { ... }
    // Getter Setter toString
}
```

然后可以使用JOOQ的API方便的将查询结果和POJO相互转化了

         // Record to POJO
         Author a = author.into(Author.class);
         List<Author> list = dsl.selectFrom(AUTHOR).fetch().into(Author.class);
         System.out.println("size " + list.size());
         
         // POJO to Record
         AuthorRecord author = dsl.newRecord(AUTHOR);
         author.from(a);

需要注意的是：
* `from`方法提供了多个重载，可以从`Map`或`Array`获取数据
* `from`方法从`Object`获取数据使用了反射的方式，因此效率会比较低。【后面会有解决办法】

### Tip 4 生成不可变POJO

如果希望这个POJO只作为数据库数据来看待，还可以使用“不可变模式”来生成对象：

```
	<generate>
		<pojos>true</pojos>
		<immutablePojos>false</immutablePojos>
	</generate>
```

这样生成的对象就不再有`Setter`方法和空构造函数了。

```
public class Author implements Serializable {
    private Integer id;
    private String  firstName;
    private String  lastName;
    public Author(Author value) { ... }
    public Author(Integer id, String firstName, String lastName) { ... }
    // Getter toString
}
```

### Tip 5 生成接口

Java提倡面向接口编程，JOOQ也提供了生成接口的配置：

```
	<generate>
		<pojos>true</pojos>
		<!-- Generate interfaces that will be implemented by records and/or pojos. 
			You can also use these interfaces in Record.into(Class<?>) and similar methods, 
			to let jOOQ return proxy objects for them. Defaults to false -->
		<interfaces>true</interfaces>
	</generate>
```

生成的接口是这样的：

```
public interface IAuthor extends Serializable {
    // Getters and Setters
    /**
     * Load data from another generated Record/POJO implementing the common interface IAuthor
     */
    public void from(test.generated.tables.interfaces.IAuthor from);
    /**
     * Copy data into another generated Record/POJO implementing the common interface IAuthor
     */
    public <E extends test.generated.tables.interfaces.IAuthor> E into(E into);
}
```

如果选择生成“接口”，则生成的`Record`和`POJO`都将继承该接口。
而之前提到的`from`方法也将会增加一个`IAuthor`的重载，并使用其中的`Getters, Setters`，从而具有更好的性能。

如果只生成接口，而不生成`Record`或`POJO`，则JOOQ会使用proxy动态代理对象的方法来达到相同的效果。

### Tip 6 生成Annotation

JOOQ可以选择生成`JPA`和`JSR-303`的注解，从而能够更好的和JPA框架配合，并提供更加强大的校验能力：

		<!-- Annotate POJOs and Records with JPA annotations for increased compatibility 
			and better integration with JPA/Hibernate, etc
			Defaults to false -->
		<jpaAnnotations>true</jpaAnnotations>
		<!-- Annotate POJOs and Records with JSR-303 validation annotations 
		    Defaults to false -->
		<validationAnnotations>true</validationAnnotations>

选择之后，以`IAuthor`为例，生成的代码将变成这样：

```
@Entity
@Table(name = "Author", schema = "Library")
public interface IAuthor extends Serializable {
    public void setId(Integer value);
    @Id
    @Column(name = "id", unique = true, nullable = false, precision = 10)
    @NotNull
    public Integer getId();
    
    public void setFirstName(String value);
    @Column(name = "first_name", length = 255)
    @Size(max = 255)
    public String getFirstName();
    
    // Getters and Setters for lastName
    // FROM and INTO
}
```

当然要想使用这些注解，需要增加对应的依赖。以`gradle`配置为例：

```
	compile group: 'javax.persistence', name: 'persistence-api', version: '1.0.2'
	compile group: 'javax.validation', name: 'validation-api', version: '1.1.0.Final'
```

### Tip 7 Builder模式

到了这一步，也许你已经在问自己，为什么没有早点认识JOOQ了，能省不少事呢。不过你知道吗，JOOQ还可以做得更多——它能生成具有Builder模式的Bean：

	<generate>
		<pojos>true</pojos>
		<interfaces>true</interfaces>
		<jpaAnnotations>true</jpaAnnotations>
		<validationAnnotations>true</validationAnnotations>
		<!-- Generate fluent setters in
		  - records
		  - pojos
		  - interfaces
		  Fluent setters are against the JavaBeans specification, but can be quite
		  useful to those users who do not depend on EL, JSP, JSF, etc.
		  Defaults to false -->
		<fluentSetters>true</fluentSetters>
	</generate>

配置`fluentSetters`为true时，`Setter`将返回自身：

```
    public Author setId(Integer id) {
        this.id = id;
        return this;
    }
```

生成的代码将可以这样使用：

```
IAuthor author = dsl.newRecord(AUTHOR);
author.setId(1).setFirstName("George").setLastName("Orwell");
```

这样代码字段设置简洁了很多。

### Tip 8 DAO

如果你是DAO模式和Spring的忠实粉丝，还可以增加以下两个配置，无缝对接Spring的集成：

```
<daos>true</daos>
<springAnnotations>true</springAnnotations>
```

生成的代码类似这样：

```
@Repository
public class AuthorDao extends DAOImpl<AuthorRecord, test.generated.tables.pojos.Author, Integer> {
    public AuthorDao() {
        super(Author.AUTHOR, test.generated.tables.pojos.Author.class);
    }
    @Autowired
    public AuthorDao(Configuration configuration) {
        super(Author.AUTHOR, test.generated.tables.pojos.Author.class, configuration);
    }
    
    @Override
    protected Integer getId(test.generated.tables.pojos.Author object) {
        return object.getId();
    }
    public List<test.generated.tables.pojos.Author> fetchById(Integer... values) {
        return fetch(Author.AUTHOR.ID, values);
    }
    ...
}
```

### Tip 9 JOOU

SQL支持无符号数字

```
CREATE TABLE `Author` (
  `id` int NOT NULL,
  `first_name` varchar(255) DEFAULT NULL,
  `last_name` varchar(255) DEFAULT NULL,
  `age` int unsigned NOT NULL,
  PRIMARY KEY (`id`)
);
```

但是java没有内建无符号数字的类型，为此JOOQ的作者提供了JOOU来处理无符号数。
例如对上面的schema生成的`age`字段类型是`UInteger`，可以使用`age.intValue()`获取对应的`int`数值。

尽管`UInteger`更加准确，但是对大多数java开发者来说，增加一个`intValue()`的调用也会觉得繁琐。
可以通过下面这个配置使POJO直接使用`int`作为字段类型：

```
    <database>
      ...      
      <unsignedTypes>false</unsignedTypes>
    </database>
```

### Tip 10 乐观锁支持

一个修改的常见场景是：用户读取了记录，作了修改之后再保存入数据库。如果这个时间里，有人修改了数据库的同一个字段，则上一个人的修改就被覆盖了。

乐观锁就是再出现这种情况时，通过抛出异常或其他方式，来提醒用户改动不安全，需要刷新并重新修改记录。

JOOQ提供了一个配置能方便地实现乐观锁，而不需要修改业务代码【详见[文档](http://www.jooq.org/doc/3.8/manual/sql-execution/crud-with-updatablerecords/optimistic-locking/)】：

```
// Properly configure the DSLContext
DSLContext optimistic = DSLContext.using(connection, SQLDialect.MYSQL,
  new Settings().withExecuteWithOptimisticLocking(true));
```

JOOQ的乐观锁模式虽然对业务逻辑代码没有侵入性，但是需要数据表提供`numeric VERSION`或`TIMESTAMP`字段，比如下表的`last_modified_time`字段：

```
CREATE TABLE `Author` (
  `id` int NOT NULL,
  `first_name` varchar(255) DEFAULT NULL,
  `last_name` varchar(255) DEFAULT NULL,
  `age` int unsigned NOT NULL,
  `last_modified_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
);
```

然后在代码生成中添加下面的配置：

```
<database>
  ...
  <!-- 支持正则，多个字段可以使用`|`分隔 -->
  <recordTimestampFields>last_modified_time</recordTimestampFields>
</database>
```

最后生成的`Table`类中将增加一个`getRecordTimestamp`方法供乐观锁使用：

    public TableField<AuthorRecord, Timestamp> getRecordTimestamp() {
        return LAST_MODIFIED_TIME;
    }


### Tip 11 修改字段命名规则

不同公司有不同的Schema规范，假设一个公司不是使用下划线命名，而是使用Camel命名法命名呢？

```
CREATE TABLE `Author` (
  `id` int NOT NULL,
  `firstName` varchar(255) DEFAULT NULL,
  `lastName` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
);
```

默认生成的字段名将变为`id`,`firstname`和`lastname`，这样就失去了可读性。

JOOQ想到了这个问题，它提供了丰富的[匹配策略](http://www.jooq.org/doc/3.8/manual-single-page/#codegen-matcherstrategy)来解决这个问题。

一般地只需要在`generator`下加上这段`strategy`配置就可以了：

```
	<strategy>
		<matchers>
			<fields>
				<field>
					<fieldMember>
					<!-- The optional transform element lets you apply a name transformation algorithm
				       to transform the actual database name into a more convenient form. Possible values are:
				       - AS_IS  : Leave the database name as it is             : MY_name => MY_name
				       - LOWER  : Transform the database name into lower case  : MY_name => my_name
				       - UPPER  : Transform the database name into upper case  : MY_name => MY_NAME
				       - CAMEL  : Transform the database name into camel case  : MY_name => myName
				       - PASCAL : Transform the database name into pascal case : MY_name => MyName -->
       					<transform>AS_IS</transform>
						<expression>$0</expression>
					</fieldMember>
				</field>
			</fields>
		</matchers>
	</strategy>
```

这样字段就变成了`id`和`lastName`

但是此时`firstName`对应的`getter`却变成了`getFirstname`，仍然不符合要求，即使像这样配置也不行：

```
	<strategy>
		<matchers>
			<fields>
				<field>
					<expression>^(.)(.+)$</expression>
					<fieldMember>
						<transform>AS_IS</transform>
						<expression>$0</expression>
					</fieldMember>
					<fieldSetter>
						<transform>AS_IS</transform>
						<expression>set$0</expression>
					</fieldSetter>
					<fieldGetter>
						<transform>AS_IS</transform>
						<expression>get$0</expression>
					</fieldGetter>
				</field>
			</fields>
		</matchers>
	</strategy>
```

对应的`getter`会变为`getfirstName`，仍然不符合要求。[事实证明](https://groups.google.com/forum/?utm_source=digest&utm_medium=email#!topic/jooq-user/Nx23jqRfShA)，没有一个简单的办法，我们需要定义自己的[命名策略](http://www.jooq.org/doc/3.8/manual/code-generation/codegen-generatorstrategy/)了，要稍微多费一点周折，但是也不是那么困难。

首先重写一个策略：

```
public class AsInDatabaseStrategy extends DefaultGeneratorStrategy {
  @Override
  public String getJavaMemberName(Definition definition, Mode mode) {
    return definition.getOutputName();
  }
  private String capitalize(final String line) {
    return Character.toUpperCase(line.charAt(0)) + line.substring(1);
  }
  @Override
  public String getJavaSetterName(Definition definition, Mode mode) {
    return "set" + capitalize(definition.getOutputName());
  }
  @Override
  public String getJavaGetterName(Definition definition, Mode mode) {
    return "get" + capitalize(definition.getOutputName());
  }
}
```

然后把自己的策略注册进来：

```
	<strategy>
		<name>test.config.AsInDatabaseStrategy</name>
	</strategy>
```

并在生成的命令中把这个类也加在类路径中，生成的结果就符合要求了！

## 应用集成

### Tip 12 与hikaricp集成管理数据源

JOOQ唯一的外部依赖就是一个JDBC连接或JDBC资源池`DataSource`；JOOQ只会使用这个连接构建`PreparedStatement`和执行SQL，并不会管理这个连接的生命周期。这样做的好处是模块功能划分做的很彻底，数据源的事情由专门的数据源库来管理`DataSource`是一个独立的模块，我们可以灵活地配置它。

应用开发需要引入一个第三方的`DatatSource`数据源来管理数据库连接；这里介绍一下[`hikaricp`](https://github.com/brettwooldridge/HikariCP)的集成方式：

Hikari提供了[多种方式](https://github.com/brettwooldridge/HikariCP#initialization)创建数据源，例如可以程序直接创建：

```
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/simpsons");
config.setUsername("bart");
config.setPassword("51mp50n");
config.addDataSourceProperty("cachePrepStmts", "true");
config.addDataSourceProperty("prepStmtCacheSize", "250");
config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");

HikariDataSource ds = new HikariDataSource(config);
DSLContext create = DSL.using(ds, SQLDialect.MYSQL);
```

### Tip 13 分库

数据库分库是生产环境中经常使用的方式，例如我们提供了一个新的`BookStore`的库，复用了`Author`表。如果我们要查询`BookStore`库，还需要重新生成`BookStore`的代码，那就太麻烦了。JOOQ提供了运行时替换库名的能力，使得在生产环境中使用多个分库变得很轻松。还以`Author`的查询为例：

```
DSLContext dsl = DSL.using(conn, SQLDialect.MYSQL);
Result<Record> result = dsl.select().from(AUTHOR).fetch();
```

对应的默认查询语句是这样的，和生成这个代码的Schema一样：

```
select `Library`.`Author`.`id`, `Library`.`Author`.`firstName`, `Library`.`Author`.`lastName`, `Library`.`Author`.`age`, `Library`.`Author`.`lastModifiedTime` from `Library`.`Author`
```

如果我们配置了动态替换

```
Settings settings =
        new Settings().withRenderMapping(new RenderMapping().withSchemata(
            new MappedSchema().withInput("Library").withOutput("BookStore")));
    
DSLContext dsl = DSL.using(conn, SQLDialect.MYSQL, settings);
Result<Record> result = dsl.select().from(AUTHOR).fetch();
```

则生成的语句就变为`BookStore`下的SQL了：

```
select `BookStore`.`Author`.`id`, `BookStore`.`Author`.`firstName`, `BookStore`.`Author`.`lastName`, `BookStore`.`Author`.`age`, `BookStore`.`Author`.`lastModifiedTime` from `BookStore`.`Author`
```

JOOQ还支持多个Schema的动态替换，以及表名的动态替换，详见[Runtime schema mapping](http://www.jooq.org/doc/latest/manual/sql-building/dsl-context/runtime-schema-mapping/)

### Tip 14 复用数据库实例

因为JOOQ的SQL是带有Schema名称的，所以对同一个IP实例中的不同数据库，JOOQ可以方便的支持查询。

这在当多个数据库共用一个数据库实例的时候，很有用，可以复用数据源配置，从而不需要为每一个库创建一个连接池，减少数据库连接资源的占用。

```
    dbsources = ImmutableMap.<String, HikariDataSource>builder()
            .put("DATABASE1", datasources1)
            .put("DATABASE2", datasources1).build();
```

### Tip 15 依赖注入

* Spring： http://www.jooq.org/doc/3.8/manual-single-page/#jooq-with-spring
* JOOQ：https://github.com/jOOQ/jOOQ/tree/master/jOOQ-examples/jOOQ-spring-guice-example

### Tip 16 其他常用操作：Batch、Curser、Transaction

* Batch： http://www.jooq.org/doc/3.8/manual/sql-execution/batch-execution/
* 游标 Curser： http://www.jooq.org/doc/3.8/manual/sql-execution/fetching/lazy-fetching/
* 事务： http://www.jooq.org/doc/3.8/manual-single-page/#transaction-management

### Tip 17 其它运行时配置

JOOQ将一些不常用的配置，放在Settings对象类管理，例如

* 是否使用乐观锁
* 是否打印JOOQ的SQL日志

具体可以参考[JOOQ文档](http://www.jooq.org/doc/3.8/manual/sql-building/dsl-context/custom-settings/)

## Unit Test

### Tip 18 jdbc mocking

现在有很多的UT工具，能够减轻UT编写的负担，JOOQ提供了JDBC的Mock工具，使我们可以不必使用第三方Mock工具就快速的进行数据库相关的UT编写。

详见 http://www.jooq.org/doc/3.8/manual-single-page/#jdbc-mocking

但是按照文档中，每一个查询都写一段代码返回数据也是冗繁的工作，如果给定一些mock数据可以这样做：

首先写一个保存模拟数据的文件放在`src/test/resources`目录下，按照 Scheme + Table的文件路径存放：

```
// 文件路径 src/test/resources/mockdata/Library/Author.txt
// id,first_name,last_name,age,last_modified_time
1,George,Orwell,46,1950-01-21 00:00:00,
```

然后在自己的`MockDataProvider`中写这样一个方法:

```
  @SuppressWarnings({"rawtypes", "unchecked"})
  public MockResult getTableRecordFromCsvFiles(Table table) throws Exception {
    DSLContext create = DSL.using(SQLDialect.MYSQL);
    Result result = create.newResult(table);

    int length = 0;
    for (String line : Resources.asCharSource(
            Resources.getResource("mockdata/" + table.getSchema().getName() + "/" + table.getName() + ".txt"),
            Charsets.UTF_8).readLines()) {
      if (line.startsWith("//") || line.trim().isEmpty())
        continue;

      String[] values = line.split(",");
      System.out.println(Arrays.asList(values));
      Record record = create.newRecord(table);
      for (int i = 0; i < table.fields().length; i++) {
        Field f = table.fields()[i];
        if (f.getType().equals(String.class)) {
          record.setValue(f, values[i]);
        } else {
          Method m = f.getType().getDeclaredMethod("valueOf", String.class);
          System.out.println(length + " : " + f + " -> " + values[i]);
          record.setValue(f, m.invoke(null, values[i]));
        }
      }
      result.add(record);
      length++;
    }
    return new MockResult(length, result);
  }
```

就可以使用这个文件的数据了：

```
      if (sql.startsWith("SELECT")){
        if(sql.contains("FROM `LIBRARY`.`AUTHOR`"))
          mock[0] = getTableRecordFromCsvFiles(AUTHOR);
        else
          throw new RuntimeException("unknow schema " + sql);
        return mock;
      }
```

当然上面的代码更像是提供数据的`Stub`，关于测试数据库，最好的策略是不要测试数据库，让JOOQ保证与数据库交互的正确性，而UT只测试业务逻辑。只有必须使用数据库层提供数据的地方使用`MockDataProvider`提供测试数据。【TODO】后面如果有时间还会补充怎样使用guice实现mocking，但是是否必要有待观察。

## Scala

首先可以修改生成器生成scala代码，效果和java代码是等效的：

```
  <generator>
    <name>org.jooq.util.ScalaGenerator</name>
  </generator>
```

然后可以使用jooq-scala提供的特性开发了：

* http://www.jooq.org/doc/3.8/manual-single-page/#jooq-and-scala
* http://www.jooq.org/doc/3.8/manual-single-page/#scala-sql-building

