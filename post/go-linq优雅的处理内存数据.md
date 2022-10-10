---
title: "【开源库推荐】go-linq优雅的处理内存数据"
date: 2022-10-10T10:04:00+08:00
draft: true
toc: true

---
## 前言
在业务开发过程中，我们经常要对从数据库中查询出的数据做处理，比如从一个切片当中取取出某个字段的切片，类似这种场景，golang官方对struct其实没有太多便捷的处理方式，大部分时候只能通过for循环来处理，那么有什么便捷又好用的方法来降低代码的复杂度，写出可读性更高的代码呢？答案是肯定的，go-linq，相信做过.net的同学早就听过这个概念，微软很早就在c#语言当中引入了linq技术，java当中，基于lamabda表达式，处理起来也很方便。
## 资料
- go-lint github地址：https://github.com/ahmetb/go-linq
- 什么是语言集成查询：https://baike.baidu.com/item/LINQ

golinq特性：
- 用 vanilla Go 编写，没有依赖关系！
- 使用迭代器模式完成惰性求值
- 支持并发安全
- 支持泛型函数，使您的代码更简洁且没有类型断言（但降低性能）
- 支持数组、切片、映射、字符串、通道和自定义集合

## 🌰说明
一个简单的struct获取age大于等于18的ID变成一个slice来给到后面的SQL执行in查询，按照一般的做法就直接开始for循环+if判断+赋值了。
```go
type Person struct {
	ID   int64
	Age  int64
	Name string
}

func getPersonIDList(persons []*Person) []int64 {
	personIDList := make([]int64, 0, len(persons))
	for _, person := range persons {
		if person.Age >= 18 {
			personIDList = append(personIDList, person.ID)
		}
	}
	return personIDList
}
```

如果是java程序是怎么做的呢？ 基于java8的特性可以很方便的通过stream -> filter -> map -> collect来获取出想要的内容也非常简洁

```java
public class Test (
  @Data
  public static class Person (
    private Long id;
    private Integer age;
    private String name;
  }
  
  public List<Long> getPersonIdList(List<Person> personList) {
    return personList.stream()
                     .filter(person -> person.age >= 18)
                     .map(Person::getId).collect(Collectors. toList());
  }
}
```
导入依赖包：
```
go get github.com/ahmetb/go-linq/v3
```
通过go-linq实现类似的效果，看起来清爽很多
```
type Person struct {
	ID   int64
	Age  int64
	Name string
}

func getPersonIDList(persons []*Person) (personIDList []int64) {
	linq.From(persons).
		WhereT(func(person *Person) bool { return person.Age >= 18 }).
		SelectT(func(person *Person) int64 { return person.ID }).ToSlice(&personIDList)
	return
}
```
PS：如果你在意性能并且list比较大就不推荐使用WhereT、SelectT，应该使用Where、Select通过interface来接受入参进行断言避免反射带来的开销，纯性能差距大改造 5 到 10 倍

## go-linq的功能及场景
官方例子：查找出书最多的作者
```
type Book struct {
		id      int
		title   string
		authors []string
	}
	var books []Book
	author := From(books).
		SelectMany( // make a flat array of authors
			func(book interface{}) Query {
				return From(book.(Book).authors)
			}).
		GroupBy( // group by author
			func(author interface{}) interface{} {
				return author // author as key
			}, func(author interface{}) interface{} {
				return author // author as value
			}).
		OrderByDescending( // sort groups by its length
			func(group interface{}) interface{} {
				return len(group.(Group).Group)
			}).
		Select( // get authors out of groups
			func(group interface{}) interface{} {
				return group.(Group).Key
			}).
		First() // take the first author
```
### ForEach
```
fruits := []string{"orange", "apple", "lemon", "apple"}
	linq.From(fruits).ForEachIndexedT(func(i int, fruit string) {
		fmt.Println(i, fruit)
	})
```
### Where
```
	fruits := []string{"apple",
		"passionfruit",
		"banana",
		"mango",
		"orange",
		"blueberry",
		"grape",
		"strawberry"}

	var query []string
	linq.From(fruits).WhereT(func(f string) bool { return len(f) > 6 }).ToSlice(&query)
	fmt.Println(query)

	numbers := []int{0, 30, 20, 15, 90, 85, 40, 75}
	var query2 []int
	linq.From(numbers).WhereIndexedT(func(index int, number int) bool { return number <= index*10 }).ToSlice(&query2)
	fmt.Println(query2)
}

type Product struct {
	Name string
	Code int
}

func Test_Distinct(t *testing.T) {
	ages := []int{21, 46, 46, 55, 17, 22, 55, 55}
	var distinctAges []int
	linq.From(ages).
		OrderBy(
			func(item interface{}) interface{} { return item },
		).
		Distinct().
		ToSlice(&distinctAges)
	fmt.Println(distinctAges)

	products := []Product{
		{Name: "apple", Code: 9},
		{Name: "orange", Code: 4},
		{Name: "apple", Code: 9},
		{Name: "Lemon", Code: 12},
	}

	var noduplicates []Product
	linq.From(products).
		DistinctByT(
			func(item Product) int { return item.Code },
		).
		ToSlice(&noduplicates)
	for _, product := range noduplicates {
		fmt.Printf("%s %d\n", product.Name, product.Code)
	}

```

### Except
```
	fruits1 := []Product{
		{Name: "apple", Code: 9},
		{Name: "orange", Code: 4},
		{Name: "apple", Code: 9},
		{Name: "Lemon", Code: 12},
	}

	fruits2 := []Product{{Name: "apple", Code: 9}}

	//Order and exclude duplicates.
	var except []Product
	linq.From(fruits1).
		ExceptByT(linq.From(fruits2),
			func(item Product) int { return item.Code },
		).
		ToSlice(&except)

	for _, product := range except {
		fmt.Printf("%s %d\n", product.Name, product.Code)
	}
```
### Intersect
```
	storel := []Product{
		{Name: "orange", Code: 4},
		{Name: "apple", Code: 9},
	}
	store2 := []Product{
		{Name: "lemon", Code: 12},
		{Name: "apple", Code: 9},
	}

	var duplicates []Product
	linq.From(storel).
		IntersectByT(linq.From(store2),
			func(p Product) int { return p.Code },
		).
		ToSlice(&duplicates)

	for _, p := range duplicates {
		fmt.Println(p.Name, p.Code)
	}

```