---

title: 极简SpringBoot指南-Chapter04-基于SpringBoot的书籍管理Web服务
date: 2021-08-10
tags:
 - Java
 - SpringBoot

---

## 仓库地址

[w4ngzhen/springboot-simple-guide: This is a project that guides SpringBoot users to get started quickly through a series of examples (github.com)](https://github.com/w4ngzhen/springboot-simple-guide)

# Chapter04-基于SpringBoot的书籍管理Web服务

<!-- more -->

从本章开始，我们将会基于SpringBoot框架，来编写一块书籍管理的应用。为了契合我们的简单教程原则，项目不会出现复杂的结构，只会有一个通用的结构。

## 初始结构

我们项目的初始结构如下：

```
base-package
    |-- controller
        |-- BookController.class
    |-- model
        |-- Book.class
    BookManagementSystemApp.class
```

## Book类

```java
public class Book {
    /**
     * 书籍ID
     */
    private String id;
    /**
     * 书籍名称
     */
    private String name;
    /**
     * 书籍价格
     */
    private double price;

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public double getPrice() {
        return price;
    }

    public void setPrice(double price) {
        this.price = price;
    }
}

```

## BookController类

```java
@RestController
@RequestMapping("books") // url："books"
public class BookController {

    private final List<Book> bookList;

    /**
     * 构造函数
     * 内部进行bookList初始化操作，便于下面的测试
     */
    public BookController() {
        int count = 3;
        this.bookList = new ArrayList<>(count);
        Random random = new Random();
        for (int idx = 0; idx < count; idx++) {

            Book book = new Book();
            book.setId(Integer.toString(idx));
            book.setName("book@" + idx);
            book.setPrice(random.nextInt(100) + 1);

            this.bookList.add(book);
        }
    }

    /**
     * GET /books
     * 返回所有的书籍信息
     */
    @GetMapping
    public List<Book> getBookList() {
        return bookList;
    }

    /**
     * GET /books/{id}
     * 根据书籍ID，得到对应的书籍信息
     * @param id 书籍ID
     * @return 书籍
     */
    @GetMapping("{id}")
    public Book getBookById(@PathVariable("id") String id) {
        Optional<Book> first = this.bookList
                .stream()
                .filter(b -> b.getId().equals(id))
                .findFirst();
        return first.orElse(null);
    }

    /**
     * POST /books
     * 添加书籍信息
     * 需要注意的是，入参Book需要添加注解@RequestBody，才能通过HTTP JSON形式传入
     * @param book 希望新增的书籍信息
     */
    @PostMapping
    public void addBook(@RequestBody Book book) {
        if (book == null) {
            System.out.println("请求数据book为空，未进行添加");
            return;
        }
        // 服务端生成ID
        String nextId = Integer.toString(this.bookList.size());
        book.setId(nextId);
        this.bookList.add(book);
    }

    /**
     * PUT /books/{id}
     * 更新指定ID书籍的信息，
     * 需要注意的是，入参Book需要添加注解@RequestBody，才能通过HTTP JSON形式传入
     * @param id 希望更新的书籍信息
     * @param book 希望更新的书籍信息
     */
    @PutMapping("{id}")
    public void updateBook(@PathVariable("id") String id, @RequestBody Book book) {
        if (book == null || id == null) {
            System.out.println("请求数据book为空或指定书籍id为空，终止更新");
            return;
        }
        Optional<Book> first = this.bookList
                .stream()
                .filter(b -> b.getId().equals(id))
                .findFirst();
        if (first.isPresent()) {
            Book exist = first.get();
            exist.setName(book.getName());
            exist.setPrice(book.getPrice());
        }
    }
    
    /**
     * DELETE /books/{id}
     * 根据书籍ID删除对应的书籍信息
     * @param id 待删除的书籍ID
     */
    @DeleteMapping("{id}")
    public void deleteBook(@PathVariable("id") String id) {
        if (id == null || id.trim().equals("")) {
            return;
        }
        Optional<Book> existBook = this.bookList
                .stream()
                .filter(b -> b.getId().equals(id))
                .findFirst();
        existBook.ifPresent(this.bookList::remove);
    }
}
```

对于该Controller，我们添加了如下的5个API：

- 获取所有的书籍信息（GET /books）
- 获取指定ID的书籍信息（GET /books/{id}）
- 增加书籍信息（POST /books）
- 更新书籍信息（PUT /books/{id}）
- 删除指定ID书籍信息（DELETE /books/{id}）

对于URL的定义形式，我们采用了REST ful规范：[[RESTful API 一种流行的 API 设计风格](https://restfulapi.cn/)](http://www.restfulapi.nl/)。

## Web应用启动

最后，我们编写一个启动类启动我们的书籍管理Web服务：

```java
@SpringBootApplication
public class BookManagementSystemApp {
    public static void main(String[] args) {
        SpringApplication.run(BookManagementSystemApp.class, args);
    }
}
```

## 功能验证

通过postman等HTTP API工具，我们可以轻松的验证我们的API的正确性。本人也把该处的postman调用文件导出存放在了项目/guide/postman/目录下，同学可以用postman导入使用。

## 仓库地址

[w4ngzhen/springboot-simple-guide: This is a project that guides SpringBoot users to get started quickly through a series of examples (github.com)](https://github.com/w4ngzhen/springboot-simple-guide)

