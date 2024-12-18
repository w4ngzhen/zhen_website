---
layout: post
title: CSharpEntityFramework与CodeFirst实践
date: 2018-09-13
tags: 
- C#
- EF
categories: 
- 技术
---

### 前言

当我们进行开发的时候,常常会用到数据库来对数据进行持久化的操作，有的时候,我们并不想要在进行代码开发的过程中，还去关注数据库的构建,表的构建等等。于是，就有了Code First模式。何为Code First模式呢？它思想就是先定义模型中的类，再通过这些类生成数据库。这种开发模式适合于全新的项目，它使得我们可以以代码为核心进行设计而不是先构造数据库。这样一来，使得我们更加关注代码的开发。在c#中，我们使用EntityFramework来实现Code First场景。

<!-- more -->

### 背景

试想一下，现在有一个图书管理项目，里面会用到Book实体类，Book会唯一编号Id、书名Title、价格Price，在数据库优先的情形下，我们可能会首先创建Book对应的表，里面创建对应于Id、Title和Price的字段，然后回到代码中继续来开发，亦或者先在代码中进行开发，然后在需要DA（数据访问）的时候创建数据库以及表结构。无论怎样，我们作为开发都要与数据库进行打交道，来回切换关注的东西，还要注意数据库的表建立的对不对，数据类型对不对等等。好在c#中有了EF这样的强大的框架以及Code First的思想。带给我们全新的开发体验。

### 实践

#### 基础配置

创建一个项目，并利用Nuget引入EntityFramework6

![nugetef6](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/nugetef6.png)

当我们引入EF时，我们发现项目下app.config其中的配置会发生更改，这个配置文件**更改的内容**就是EF为我们创建的，也是我们配置数据库连接的地方。

回到项目中，接下来我们创建Book实体类，为其添加Id、Title以及Price属性，同时使用特性在属性以及类名上标注该实体类在数据库中的体现方式：

```c#
namespace CodeFirstDemo
{
    [Table("Book")]
    public class Book
    {
        private Guid id;
        public Guid Id
        {
            get
            {
                id = id == null ? Guid.NewGuid() : id;
                return id;
            }
        }
        [Required]
        public string Title { get; set; }
        [Required]
        public double Price { get; set; }
    }
}
```

使用Table特性来表名该实体类Book将对应数据库中的book表（不需要此刻已经有Book表），使用[Required]特性来表明字段是否可为空，此外，由于EF默认将Id属性视为主键，所以无需使用[Key]特性来指明上面的Id为主键。

接下来，我们需要使用继承EF的DbContext来构建数据库上下文类，我们直接使用VS自带生成工具即可生成对应的数据库上下文模型：

![bookdbdemo](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/bookdbdemo.png)
![genguide](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/genguide.png)

点击完成后我们就得到了如下的一个配置类

```c#
 public class BookDbDemo : DbContext
    {
        //您的上下文已配置为从您的应用程序的配置文件(App.config 或 Web.config)
        //使用“BookDbDemo”连接字符串。默认情况下，此连接字符串针对您的 LocalDb 实例上的
        //“CodeFirstDemo.BookDbDemo”数据库。
        // 
        //如果您想要针对其他数据库和/或数据库提供程序，请在应用程序配置文件中修改“BookDbDemo”
        //连接字符串。
        public BookDbDemo()
            : base("name=BookDbDemo")
        {
        }

        //为您要在模型中包含的每种实体类型都添加 DbSet。有关配置和使用 Code First  模型
        //的详细信息，请参阅 http://go.microsoft.com/fwlink/?LinkId=390109。

        // public virtual DbSet<MyEntity> MyEntities { get; set; }
    }

    //public class MyEntity
    //{
    //    public int Id { get; set; }
    //    public string Name { get; set; }
    //}
```
一个十分简单的数据库上下文类就建立完成。

正如生成的DbContext所说：“为您要在模型中包含的每种实体类型都添加 DbSet。”，我们在该类中添加如下的DbSet属性，并将注释删除，更加直观的看一看当前的结构：

```c#
public class BookDbDemo : DbContext
{
    public BookDbDemo()
        : base("name=BookDbDemo")
    {
    }
    public virtual DbSet<Book> Books { get; set; }
}
```
在这个类中，我们声明了一个DbSet属性books，这就对应了数据库中的book表。换句话说，继承了DbContext的类就对应了某一个数据库，其连接属性由配置文件中的连接配置决定，并在DbContext中设置进去（基类构造函数设置），这个DbContext中的所有DbSet就对应到数据库中的表。

注意到，构造函数调用了基类构造函数，传入了"name=BookDbDemo"字符串，这个字符串就是指app.config配置文件中的数据库连接名，然后我们查看App.config文件，发现vs已经为我们生成了一个连接字符串节点：

```xml
  <connectionStrings>
    <add name="BookDbDemo" connectionString="data source=(LocalDb)\MSSQLLocalDB;initial catalog=CodeFirstDemo.BookDbDemo;integrated security=True;MultipleActiveResultSets=True;App=EntityFramework" providerName="System.Data.SqlClient" />
  </connectionStrings>
```
但是这还段配置还是有点问题，首先连接地址数据库用户名等等都不对，更主要的是，我是MySQL数据库，怎么能用SQL呢，所以，我们要解决EF进行MySQL的连接问题。

#### 使用EF进行MySQL数据库连接配置

如果使用ado.net链接mysql数据库则只需要MySql.Data.dll，即使用Nuget安装Mysql.Data就行了，并不需要安装mysql-connector-net驱动程序；

如果使用EF的话一般来说需要安装mysql-connector-net驱动程序；

其中mysql-connector-net驱动程序安装目录包含了
MySql.Data.dll；
MySql.Data.Entity.EF5.dll；
MySql.Data.Entity.EF6.dll；
MySql.Fabric.Plugin.dll；
MySql.Web.dll；

注意：就算把mysql-connector-net安装目录下所有的类库都拷到bin目录而不在应用环境上安装mysql-connector-net驱动程序，EF代码还是会报错，因为mysql-connector-net安装不仅装了各种dll类库，还在机器上.net环境的全局machine.config里加过如下配置：
```xml
<system.data>
  <DbProviderFactories>
  <add name="MySQL Data Provider"
  invariant="MySql.Data.MySqlClient"
  description=".Net Framework Data Provider for MySQL"
  type="MySql.Data.MySqlClient.MySqlClientFactory, MySql.Data, Version=6.9.6.0, Culture=neutral,        PublicKeyToken=c5687fc88969c44d" />
</DbProviderFactories>
</system.data>
```
所以要使用EF必须在站点服务器安装mysql-connector-net启动程序（数据库所在的服务器不需要安装，只需要c#程序所运行的电脑需要安装）**或者**在应用的配置文件（app.config或web.config）里configuration节点下加上上面的配置。

这里，为了我们机器环境的纯净，我们使用nuget安装对应项目需要的库（MySql.Fabric.Plugin.dll和MySql.Web.dll这两个库如果没有需要不用安装）到项目中：

![mysqldata](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/mysqldata.png)
![mysqldataentity](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/mysqldataentity.png)

**务必注意！MySql.Data和MySql.Data.Entity必须要保持相同的版本！否则会出现 “找到的程序集清单定义与程序集引用不匹配。 (异常来自 HRESULT:0x80131040)” 的错误，所以这里的Mysql.Data版本我并没有选择最新的**

并且在配置文件中加如上述system.data配置，此时app.config配置主要有如下的变化：
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
	...
  </configSections>
  <startup>
    ...
  </startup>
  <entityFramework>
    <defaultConnectionFactory 
        ...
    </defaultConnectionFactory>
    <providers>
      ...
      <!-- 新增了 MySql.Data.MySqlClient Provider -->
      <provider invariantName="MySql.Data.MySqlClient" type="MySql.Data.MySqlClient.MySqlProviderServices, MySql.Data.Entity.EF6, Version=6.10.8.0, Culture=neutral, PublicKeyToken=c5687fc88969c44d"/>
    </providers>
  </entityFramework>
  <connectionStrings>
    <add name="BookDbDemo" connectionString="data source=(LocalDb)\MSSQLLocalDB;initial catalog=CodeFirstDemo.BookDbDemo;integrated security=True;MultipleActiveResultSets=True;App=EntityFramework" providerName="System.Data.SqlClient" />
  </connectionStrings>
  <!-- 引入MySql.Data写入的 -->
  <runtime>
    <assemblyBinding xmlns="urn:schemas-microsoft-com:asm.v1">
      <dependentAssembly>
        <assemblyIdentity name="MySql.Data" publicKeyToken="c5687fc88969c44d" culture="neutral" />
        <bindingRedirect oldVersion="0.0.0.0-6.9.6.0" newVersion="6.9.6.0" />
      </dependentAssembly>
    </assemblyBinding>
  </runtime>
  <!-- 手动新增system.data配置节点 -->
  <system.data>
    <DbProviderFactories>
      <add name="MySQL Data Provider"
           invariant="MySql.Data.MySqlClient"
           description=".Net Framework Data Provider for MySQL"
           type="MySql.Data.MySqlClient.MySqlClientFactory, MySql.Data, Version=6.9.6.0, Culture=neutral, PublicKeyToken=c5687fc88969c44d" />
    </DbProviderFactories>
  </system.data>
</configuration>
```
PS：实际上本人在实践过程中发现，引入Mysql.Data后写如的节点（见上面xml）作用似乎和手动新增的一样，没有求证差别。

#### 实际数据库配置

完成EF的Mysql连接环境配置后，最基础的数据库还是需要建立的，所以去数据库创建一个名为bookdbdemo的数据库，按道理来说，我们只需要在这个地方触碰到数据库，况且这还是DBA的事情。创建好的数据库如下：

![mysqldb](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/mysqldb.png)

于是，我们将其中的连接字符串connectionString内容修改为我们当前的数据库连接字符串：
```xml
  <connectionStrings>
    <add name="BookDbDemo" connectionString="Data Source=localhost; Database=bookdbdemo; User ID=root; Password=;" providerName="MySql.Data.MySqlClient"/>
  </connectionStrings>

注意：providerName写的是在引入MySql.Data.Entity之后在app.config中出现的
<provider invariantName="MySql.Data.MySqlClient" type="MySql.Data.MySqlClient.MySqlProviderServices, MySql.Data.Entity.EF6, Version=6.10.8.0, Culture=neutral, PublicKeyToken=c5687fc88969c44d">节点
```
完成上面的配置之后，我们接下来就要使用EF最强大的Migration数据库迁移功能。

#### EF Database Migration EF数据库迁移

首先启用迁移功能。在Nuget命令行中输入：Enable-Migrations（有个s，注意）
```shell
PM> enable-migrations
```
注意：如果此处提示：具有固定名称“MySql.Data.MySqlClient”的 ADO.NET 提供程序未在计算机或应用程序配置文件中注册或无法加载。需要在上下文继承类中（此处就是BookDbDemo）上添加特性[DbConfigurationType(typeof(MySql.Data.Entity.MySqlEFConfiguration))]
```c#
...
[DbConfigurationType(typeof(MySql.Data.Entity.MySqlEFConfiguration))]
public class BookDbDemo : DbContext
{
    ...
}
```

启动迁移功能后，Nuget命令行提示：
```shell
PM> enable-migrations
正在检查上下文的目标是否为现有数据库...
已为项目 CodeFirstDemo 启用 Code First 迁移。
```

此处提示我们，EF的数据迁移功能已经启用，在项目中我们会发现创建了一个名为Migtaions的文件夹，里面还存在一个Configuration配置类，这个类中，我们需要将AutomaticMigrationsEnabled设置为true，即启用自动迁移功能

```c#
public Configuration()
{
    AutomaticMigrationsEnabled = true;
}
```

此时，我们的数据库还没有创建的任何的表。

接下来，我们使用Add-Migtaion XXX命令来添加一个变更模块。

#### 初始化以及创建表

```shell
PM> add-migration InitDb
正在为迁移“InitDb”搭建基架。
此迁移文件的设计器代码包含当前 Code First 模型的快照。在下一次搭建迁移基架时，将使用此快照计算对模型的更改。如果对要包含在此迁移中的模型进行其他更改，则您可通过再次运行“Add-Migration InitDb”重新搭建基架。
```
注意后面的命名，InitDb只是我们取的名字，为了区分对数据库进行的变更，这里是我们第一次构建，所以我取名为InitDb，完成该命令后，你会发现Migrations文件夹下出现了一个以你迁移模块创建时刻+下划线+刚刚迁移模块的命名的类文件：

![InitDb](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/InitDb.png)

其类文件内容如下：

```c#
public partial class InitDb : DbMigration
    {
        public override void Up()
        {
            CreateTable(
                "dbo.Book",
                c => new
                    {
                        Id = c.Guid(nullable: false),
                        Title = c.String(nullable: false),
                        Price = c.Double(nullable: false),
                    })
                .PrimaryKey(t => t.Id);
            
        }
        
        public override void Down()
        {
            DropTable("dbo.Book");
        }
    }
```

我们可以看到里面重写了DbMigration的Up和Down方法，同时从方法中的代码我们能大致推测，Up方法就是发生的变更，这里会创建表Book（dbo是EF的一些配置语法），设置表字段以及设置主键；而Down方法就是回滚操作，将表Book删除。

但是此时，我们数据库中的表还是没有发生变化，因为我们现在添加了变更模块，只是根据当前的代码来的，要将变更同步到数据库，需要Update-Database命令（添加-Versbose方便我们查看数发生执行的SQL语句）：

```shell
PM> update-database -verbose
Using StartUp project 'CodeFirstDemo'.
Using NuGet project 'CodeFirstDemo'.
指定“-Verbose”标志以查看应用于目标数据库的 SQL 语句。
目标数据库为: “bookdbdemo”(DataSource: localhost，提供程序: MySql.Data.MySqlClient，来源: Configuration)。
正在应用显式迁移: [201809131330306_InitDb]。
正在应用显式迁移: 201809131330306_InitDb。
create table `Book` (`Id` CHAR(36) BINARY default ''  not null ,`Title` longtext not null ,`Price` double not null ,primary key ( `Id`) ) engine=InnoDb auto_increment=0
create table `__MigrationHistory` (`MigrationId` nvarchar(150)  not null ,`ContextKey` nvarchar(300)  not null ,`Model` longblob not null ,`ProductVersion` nvarchar(32)  not null ,primary key ( `MigrationId`) ) engine=InnoDb auto_increment=0
INSERT INTO `__MigrationHistory`(
`MigrationId`, 
`ContextKey`, 
`Model`, 
`ProductVersion`) VALUES (
'201809131330306_InitDb', 
'CodeFirstDemo.Migrations.Configuration', 
#中间有一大段经过摘要的字符串，应该是变更的摘要
, 
'6.2.0-61023');
正在运行 Seed 方法。
```
我们可以从输出中很容易的看到执行了创建book数据库的sql语句，以及创建了一个MigrationHistory表，这个表就是记录了数据库迁移的一些摘要，供我们以后来进行回退操作。

完成了迁移之后，查看数据库：

![initcomplete](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/initcomplete.png)
![tabledetail](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/tabledetail.png)

我们可以看到，表及其结构按照我们预期创建成功了。

#### 删除表

为了证明EF再删除表的时候，并不会影响其他的表，我再次利用创建了一个新的实体类EBook并更新DbContext以及进行EF迁移功能：
```c#
namespace CodeFirstDemo
{
    [Table("EBook")]
    public class EBook
    {
        private Guid id;
        public Guid Id
        {
            get
            {
                id = id == null ? Guid.NewGuid() : id;
                return id;
            }
            set
            {
                id = value;
            }
        }
        [Required]
        public string EBookTitle { get; set; }
    }
}
```

更新DbContext：

```c#
[DbConfigurationType(typeof(MySql.Data.Entity.MySqlEFConfiguration))]
public class BookDbDemo : DbContext
{
    public BookDbDemo()
    : base("name=BookDbDemo")
    {
    }
    public virtual DbSet<Book> Books { get; set; }
    // 新增的DbSet
    public virtual DbSet<EBook> EBooks { get; set; }
}
```

进行EF数据库迁移，此时我们不需要再次进行启用迁移功能了，而是使用add-migration检测并增加行的变更模块：

```shell
PM> add-migration AddEBookEntity
正在为迁移“AddEBookEntity”搭建基架。
此迁移文件的设计器代码包含当前 Code First 模型的快照。在下一次搭建迁移基架时，将使用此快照计算对模型的更改。如果对要包含在此迁移中的模型进行其他更改，则您可通过再次运行“Add-Migration AddEBookEntity”重新搭建基架。
```
此时Migrations文件夹下面又增加了新的类文件：

![addebookentity](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/addebookentity.png)

其内容和前面大致，创建表以及回滚，这里不再展示。之后我们再次使用update-database命令将变更更新到数据库中，得到当前的数据库内容：

![afteraddtable](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/afteraddtable.png)

此时我们将book表中填充一些数据：

![booktablecontent](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/booktablecontent.png)

然后，我们将DbContext中的DbSet<EBook>属性删除，再次进行迁移：

![delebook](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/delebook.png)

进行更新以后，我们可以看到Ebook表已经删除了，但是book表内容没有发生任何变化：

![afterdelebook](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/afterdelebook.png)

#### 变更属性

有的时候，我们可能很少会将实体类，更多的是对现有实体类中进行属性的增加、删除以及修改，变相的，对数据库中的表的字段进行增加、删除和修改：

##### 增加属性

现在，我们想要将Book实体类中增加一个Abstract摘要属性，它会影响我们上面我们已有的数据吗？试一试就知道了。

首先，在Book类中增加这一属性，不设置Required特性：

```c#
    ...
    [Required]
    public string Title { get; set; }
    // 增加的摘要属性
    public string Abstract { get; set; }

    [Required]
    public double Price { get; set; }
    ...
```
此时我们增加变更模块：
```shell
PM> add-migration AddNewProp
正在为迁移“AddNewProp”搭建基架。
此迁移文件的设计器代码包含当前 Code First 模型的快照。在下一次搭建迁移基架时，将使用此快照计算对模型的更改。如果对要包含在此迁移中的模型进行其他更改，则您可通过再次运行“Add-Migration AddNewProp”重新搭建基架。
```
我们可以看到变更模块类：
```c#
public partial class AddNewProp : DbMigration
{
    public override void Up()
    {
        AddColumn("dbo.Book", "Abstract", c => c.String(unicode: false));
    }

    public override void Down()
    {
        DropColumn("dbo.Book", "Abstract");
    }
}
```
从这个类的Up方法中，我们看，他调用了AddColumn增加列的方法。而Down回退方法则调用了DropColumn删除列的方法。当我们执行update-database命令有什么效果呢？

```shell
PM> update-database -verbose
Using StartUp project 'CodeFirstDemo'.
Using NuGet project 'CodeFirstDemo'.
指定“-Verbose”标志以查看应用于目标数据库的 SQL 语句。
目标数据库为: “bookdbdemo”(DataSource: localhost，提供程序: MySql.Data.MySqlClient，来源: Configuration)。
正在应用显式迁移: [201809131406177_AddNewProp]。
正在应用显式迁移: 201809131406177_AddNewProp。
alter table `Book` add column `Abstract` longtext 
INSERT INTO `__MigrationHistory`(
...
）
```
重点看到这句SQL：“alter table `Book` add column `Abstract` longtext ”，这句SQL就是对我们表添加了一个字段，类型为longtext。我们进入数据库中，看一看变化：

![afteraddprop](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/afteraddprop.png)

可以看到数据库中其他字段的值都没有发生变化，仅仅多出了这个字段，同时符合我们设置的可以为空的预期

![addpropdetail](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/addpropdetail.png)

##### 删除属性

删除与增加同理，我们直接将Book实体类的属性删除，然后增加变更，最后同步更新变更到数据库。这里我们将Price和Abstract属性都删除，变更模块如下：
```c#
public partial class RemoveProp : DbMigration
{
    public override void Up()
    {
        DropColumn("dbo.Book", "Abstract");
        DropColumn("dbo.Book", "Price");
    }

    public override void Down()
    {
        AddColumn("dbo.Book", "Price", c => c.Double(nullable: false));
        AddColumn("dbo.Book", "Abstract", c => c.String(unicode: false));
    }
}
```
得到如下的结果：

![afterdel2prop](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/afterdel2prop.png)

##### 重命名属性

重命名比起前面的增删属性有所不同，就当前情形为例子，当前Book类属性如下：
```c#
namespace CodeFirstDemo
{

    [Table("Book")]
    public class Book
    {
        private Guid id;
        public Guid Id
        {
            get
            {
                id = id == null ? Guid.NewGuid() : id;
                return id;
            }
            set
            {
                id = value;
            }
        }
        [Required]
        public string Title { get; set; }

    }
}
```
对应数据库为
![beforerename](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/beforerename.png)

此时如果我们想要修改属性名，我们将Book中的Title属性改为Name属性：
```c#
...
[Required]
public string Name { get; set; }
...
```
然后执行Add-Migration命令：
```c#
PM> add-migration ChangePropName
```
我们看到变更类的内容为：
```c#
public partial class ChangePropName : DbMigration
{
    public override void Up()
    {
        AddColumn("dbo.Book", "Name", c => c.String(nullable: false, unicode: false));
        DropColumn("dbo.Book", "Title");
    }

    public override void Down()
    {
        AddColumn("dbo.Book", "Title", c => c.String(nullable: false, unicode: false));
        DropColumn("dbo.Book", "Name");
    }
}
```
我们发现，只要更新到数据库，EF会在表中先添加一个字段Name，然后删除字段Title，很显然，这样更新，会将我们现有的数据清空。如何不造成这样的情况呢？其实我们首先知道，在进行Update-database的时候，EF框架会执行Up方法，同时，AddColumn、DropColumn很显然是DbMigration这个类中的方法，我们找一找看有没有重命名的方法呢。令我们欣喜的是，有：
```c#
public override void Up()
{
    //AddColumn("dbo.Book", "Name", c => c.String(nullable: false, unicode: false));
    //DropColumn("dbo.Book", "Title");
    RenameColumn("dbo.Book", "Title", "Name");
}
```
注释掉Add和DropColumn，使用RenameColumn，填入表、原字段名、新字段名；接下来我们执行Update-Database -Verbose：

```shell
PM> UPDATE-DATABASE -VERBOSE
...
Fatal error encountered during command execution. ---> MySql.Data.MySqlClient.MySqlException (0x80004005): Parameter '@columnType' must be defined.
...
```
我们发现报了一个错误，实际上解决方法在链接字符串中加入这样一句话“;Allow User Variables=True”就可以了，即如下：
```xml
  <connectionStrings>
    <add name="BookDbDemo" connectionString="Data Source=localhost; Database=bookdbdemo; User ID=root; Password=;Allow User Variables=True" providerName="MySql.Data.MySqlClient"/>
  </connectionStrings>
```
这样一来，再次运行，不报错。查看数据库：

![afterrename](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-09-13-EF/afterrename.png)
