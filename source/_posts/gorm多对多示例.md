---
title: gorm多对多示例
date: 2019-02-07 18:39:09
tags: 
  - gorm
  - 数据库
---
gorm的多对多操作，真是一大堆的坑，官网的文档说得很模糊，也没有完整的例子， 中文的网络资源对这部分的解释很少。作为新手的我，在这上面卡了很长一段时时间，经过艰苦的摸索，终于爬坑成功。在这里记录下心得。

1.多对多关系的定义

```
type Student struct {
	gorm.Model
	Code string
	Name string
	Books []Book `gorm:"many2many:student_books;"`
}

type Book struct {
	gorm.Model
	Name string
	Students []Student `gorm:"many2many:student_books;"`
}
```
切记`gorm:`跟`many2many`之间**不能有空格**，否则联结表不会生成。定义了模型以后，只要调用`db.AutoMigrate(&Student{}, &Book{})`会生成三张表。
```
mysql> show tables;
+-----------------------------------+
| Tables_in_gorm                    |
+-----------------------------------+
| books                             |
| student_books                     |
| students                          |
+-----------------------------------+
```

2.多对多数据的插入
有两种方式:
2.1 直接在数据定义的时候指定。
```
student3 := Student{Code: "000003", Name: "王五" }
db.Save(&student3)
book2 := Book{
	Name: "神雕侠侣", Students: []Student {
		student3,
	},
}
```
这里需要注意，指定的数据必须是数据库有的数据，也就是保存过，或者查找出来的数据结构。如果按照下面这样的写法：
```
book2 := Book{
	Name: "神雕侠侣", Students: []Student {
		{Code: "000003", Name: "王五" },
	},
}
```
将会在student表中创建一个新项目，并不会用数据库已有的数据来建立联结。

所以下面这样插入也是可以的。
```
var studentlist []Student
db.Table("students").Where("id = 1").Or("id = 2").Find(&studentlist)

book1 := Book{Name: "笑傲江湖", Students: studentlist}
```

2.2 使用关联模式的append插入
    
```
db.Model(&bookQ).Association("Students").Append(Student{Code: "000005", Name: "西门吹雪"})
```
这里面也有个坑，在下面我再介绍。

经过上面这样数据插入以后，联结表的关系会自动的创建出来。

3.关联模式
 
关联模式最大的坑跟上面一样，就是Model的参数，必须是数据库里的数据，否则就是crash

当表建立完成以后，我们要操作的某一个项目的联系，可以像下面这样：
```
var bookQ Book
db.Table("books").Where("id = 1").First(&bookQ)
db.Model(&bookQ).Association("Students").Find(&studentlist)
```
这样所有第一本书借阅的同学都出来了。

要增加一个同学给这本书
```
db.Model(&bookQ).Association("Students").Append(student3)
```
这里也要记得student3必须是数据库保存，或者查找的数据。

删除某个关系
```
db.Model(&bookQ).Association("Students").Delete(student3)
```

完整的代码如下：
```
package main

import (
	"github.com/jinzhu/gorm"
	_ "github.com/jinzhu/gorm/dialects/mysql"
	"fmt"
)

type Student struct {
	gorm.Model
	Code string
	Name string
	Books []Book `gorm:"many2many:student_books;"`
}

type Book struct {
	gorm.Model
	Name string
	Students []Student `gorm:"many2many:student_books;"`
}

func main() {
	db := Connect()
	defer db.Close()
	db.LogMode(true)
	purgeDB(db)
	db.AutoMigrate(&Student{}, &Book{})

	student1 := Student{Code: "000001", Name: "张三"}
	student2 := Student{Code: "000002", Name: "李四"}
	student3 := Student{Code: "000003", Name: "王五" }
	db.Save(&student1)
	db.Save(&student2)
	db.Save(&student3)

	var studentlist []Student
	db.Table("students").Where("id = 1").Or("id = 2").Find(&studentlist)

	book1 := Book{Name: "笑傲江湖", Students: studentlist}
	book2 := Book{
		Name: "神雕侠侣", Students: []Student {
			student3,
		},
	}

	db.Save(&book1)
	db.Save(&book2)

	var student Student
	db.Table("students").Where("id = 1").First(&student)
	book := []Book{}
	db.Preload("Students").Find(&book)
	fmt.Println(book)

	db.Model(&student).Related(&book, "Books")
	fmt.Println(book)
	db.Model(&student).Association("Books").Find(&book)
	fmt.Println(book)

	var bookQ Book
	db.Table("books").Where("id = 1").First(&bookQ)
	db.Model(&bookQ).Association("Students").Find(&studentlist)
	fmt.Println(studentlist)

	db.Model(&bookQ).Association("Students").Append(student3)
	db.Model(&bookQ).Association("Students").Append(Student{Code: "000005", Name: "西门吹雪"})

	db.Model(&bookQ).Association("Students").Delete(student3)
	db.Model(&bookQ).Association("Students").Clear()
}

func purgeDB(db *gorm.DB) {
	if db.HasTable(&Student{}) {
		db.DropTable(&Student{})
	}
	if db.HasTable(&Book{}) {
		db.DropTable(&Book{})
	}
}

```