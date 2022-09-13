# 闭包与defer

## 1.闭包

闭包 ： 一个函数与其相关的引用环境组合的一个实体，其实可以理解为面向对象中类中的属性与方法。
如代码块中，函数function的返回值(匿名函数)与变量n就是1个闭包。
**该匿名函数就相当于类中的方法 变量n相当于类中的属性**

``` go
// 无形参 返回值是该匿名函数
func function() func(int) int {
    var n int = 10           // 相当于类属性
    return func(x int) int { //匿名函数
        x = x + n
        
        return x
    }
}
var f func(int) int = function()
fmt.Println(f(1)) // 11 
fmt.Println(f(2)) // 13
```

再举几个例子：

```go
//示例1
func adder2(x int) func(int) int {
	return func(y int) int {
		x += y
		return x
	}
}
func main() {
	var f = adder2(10)
	fmt.Println(f(10)) //20
	fmt.Println(f(20)) //40
	fmt.Println(f(30)) //70

	f1 := adder2(20)
	fmt.Println(f1(40)) //60
	fmt.Println(f1(50)) //110
}
//示例2
func makeSuffixFunc(suffix string) func(string) string {
	return func(name string) string {
		if !strings.HasSuffix(name, suffix) {
			return name + suffix
		}
		return name
	}
}

func main() {
	jpgFunc := makeSuffixFunc(".jpg")
	txtFunc := makeSuffixFunc(".txt")
	fmt.Println(jpgFunc("test")) //test.jpg
	fmt.Println(txtFunc("test")) //test.txt
}
```

## 2.defer

1. defer 是 Go 语言提供的一种用于注册延迟调用的机制，每一次 defer 都会把函数压入栈中，当前函数返回前再把延迟函数取出并执行。

   defer 定义的函数会先进入一个栈，函数 return 前，会按先进后出（FILO）的顺序执行。也就是说最先被定义的 defer 语句最后执行。

2. defer 语句定义时，对 **外部变量的引用** 是有两种方式的，分别是作为 **函数参数** 和作为 **闭包引用**。

   - 作为 **函数参数**，则在 defer **定义时** 就把值传递给 defer，并被 **缓存** 起来；
   - 作为 **闭包引用** 的话，则会在 defer 函数真正调用时根据整个上下文确定当前的值。

下面就分别对这两种情况举例子。

**情况一：**

```go
func trace(str string) string {
    fmt.Println("entering " + str)
    return str
}
func leave(str string) {
    fmt.Println("leaving " + str)
}
func point() {
    defer leave(trace("point"))
    fmt.Println("in point")
}
func main() {
    point()
}

//输出结果：
//entering point
//in point
//leaving point
```

这是第一种情况，defer的函数接受的参数在它入栈的时候就被缓存下来了。

再举个例子：

```go
func main() {
    a := 1
    b := 2
    defer calc("1", a, calc("10", a, b))
    a = 0
    defer calc("2", a, calc("20", a, b))
    b = 1
}

func calc(index string, a, b int) int {
    ret := a + b
    fmt.Println(index, a, b, ret)
    return ret
}

//10 1 2 3
//20 0 2 2
//2 0 2 2
//1 1 3 4
```

**情况二：**

要完全理解第二条规则，需要了解 `return` 与 `defer` 是怎么运行的。

函数内的 `return xxx` 并不是一个原子执行的返回：即不是先执行 `return xxx` 再执行 `defer`，也不是先执行 `defer` 再执行 `return xxx`。而是将 `return xxx` 拆分开来，经过编译后执行过程如下：

```
1. 返回变量 = xxx
2. 调用 defer 函数（有可能更新返回变量的值）
3. return 返回变量。
```

```go
1.
func f1() (r int) {
    defer func() {
        r++
    }()
    return 0
}

2.
func f2() (r int) {
    t := 5
    defer func() {
        t = t + 5
    }()
    return t
}

3.
func f3() (r int) {
    defer func(r int) { // 作为函数参数传入 defer 函数
        r = r + 5 
    }(r)
    return 1
}
拆解：
1.r = 0 // 1. 赋值
func() { // 2. 运行 defer 函数 r++，r = 1
    r++
}()
return r // 3. return，即返回结果为 1

2.r = t (= 5) // 1. 赋值，r 取值 5
func() { // 2. 执行 defer 函数，执行后 t = 10，但 r = 5
    t = t + 5
}()
return r // 3. return r，即返回 5

3.r = 1 // 1. 赋值， r 取值 1
func(r int) { // 2. 执行 defer 函数，但作为函数参数传入（缓存值为0）
    r = r + 5 // 执行后 r = 0 + 5 = 5，但这是局部变量，函数外仍是 1
}(r)
return r // 3. return r， 即返回 1
```

**踩坑点：**

```go
func increaseA() int {
    var i int
    defer func() {
        i++
    }()
    return i
}
```

注意，上面这段代码的返回值是匿名的，所以结果返回0。

现在我们再以2个例子来做总结和巩固：

```go
type Person struct {
    age int
}

func main() {
    person := &Person{28}

    // 1. 
    defer fmt.Println(person.age)

    // 2.
    defer func(p *Person) {
        fmt.Println(p.age)
    }(person)  

    // 3.
    defer func() {
        fmt.Println(person.age)
    }()

    person.age = 29
}
```

参考答案及解析：29 29 28。变量 person 是一个指针变量 。

1.person.age 此时是将 28 当做 defer 函数的参数，会把 28 缓存在栈中，等到最后执行该 defer 语句的时候取出，即输出 28；

2.defer 缓存的是结构体 Person{28} 的地址，最终 Person{28} 的 age 被重新赋值为 29，所以 defer 语句最后执行的时候，依靠缓存的地址取出的 age 便是 29，即输出 29；

3.闭包引用，输出 29；

又由于 defer 的执行顺序为先进后出，即 3 2 1，所以输出 29 29 28。

```go
type Person struct {
    age int
}

func main() {
    person := &Person{28}

    // 1.
    defer fmt.Println(person.age)

    // 2.
    defer func(p *Person) {
        fmt.Println(p.age)
    }(person)

    // 3.
    defer func() {
        fmt.Println(person.age)
    }()

    person = &Person{29}
}
```

参考答案及解析：29 28 28。这道题在第 19 天题目的基础上做了一点点小改动，前一题最后一行代码

person.age = 29 是修改引用对象的成员 age，这题最后一行代码 person = &Person{29} 是修改引用对象本身，来看看有什么区别。

1.person.age 这一行代码跟之前含义是一样的，此时是将 28 当做 defer 函数的参数，会把 28 缓存在栈中，等到最后执行该 defer 语句的时候取出，即输出 28；

2.defer 缓存的是结构体 Person{28} 的地址，这个地址指向的结构体没有被改变，最后 defer 语句后面的函数执行的时候取出仍是 28；

3.闭包引用，person 的值已经被改变，指向结构体 Person{29}，所以输出 29.

由于 defer 的执行顺序为先进后出，即 3 2 1，所以输出 29 28 28。

**终极神题！**

```go
func F(n int) func() int {
    return func() int {
        n++
        return n
    }
}

func main() {
    f := F(5)
    defer func() {
        fmt.Println(f())//闭包第三次执行（defer闭包引用）
    }()
    defer fmt.Println(f())//闭包第一次执行（defer参数传递）
    i := f()
    fmt.Println(i)//闭包第二次执行
}
//7 6 8
```