# [返回](/)

# go语法

## 基本语法

编译型、强类型（隐式类型）、动态类型

- 非驼峰命名，首字母大写

- 类型声明在后

- 短变量声明：`a := 1`，只能用在函数内

- 命令行参数：os包，os.Args，是一个切片

- 简单io

  - bufio包：

    - `input := bufio.NewScanner(os.Stdin)`
    - `input.Scan()，读下一行，返回是否有下一行`
    - `input.Text()，显示读取内容`

  - `f, err := io.Open(fileName)`

  - `data, err := ioutil.ReadFile(filename)`：一次性读取全部文件

  - 格式化

    ```go
    %d          十进制整数
    %x, %o, %b  十六进制，八进制，二进制整数。
    %f, %g, %e  浮点数： 3.141593 3.141592653589793 3.141593e+00
    %t          布尔：true或false
    %c          字符（rune） (Unicode码点)
    %s          字符串
    %q          带双引号的字符串"abc"或带单引号的字符'c'
    %v          变量的自然形式（natural format）
    %T          变量的类型
    %%          字面上的百分号标志（无操作数）
    ```

- 返回值可命名

  ```go
  func foo(x int) int{
      return x + 1
  }
  
  // 命名返回值
  func foo(x int) (ret int){
      ret = x + 1
      // 可以直接返回，返回的是ret
      return
  }
  ```

- 基本数据类型

  ```go
  bool
  
  string
  
  int  int8  int16  int32  int64
  uint uint8 uint16 uint32 uint64 uintptr
  
  byte // uint8 的别名
  
  rune // int32 的别名
      // 表示一个 Unicode 码点
  
  float32 float64
  
  complex64 complex128
  ```

  **rune**

  

- defer关键字：推迟执行，defer的函数先完成求值，但直到外层函数返回时才会被调用

  ```go
  package main
  
  import "fmt"
  
  func main() {
  	defer fmt.Println("world")
  
  	fmt.Println("hello")
  }
  // answer is “hello world”
  ```

  本质是压入函数栈

  ```go
  package main
  
  import "fmt"
  
  func main() {
  	fmt.Println("counting")
  
  	for i := 0; i < 10; i++ {
  		defer fmt.Println(i)
  	}
  
  	fmt.Println("done")
  }
  // 9 8 7 6 5 4 3 2 1 done
  ```

## 特殊结构

### 指针

指针存放数据的地址，可通过指针对数据进行访问&修改

```go
package main

import "fmt"

func main() {
	i, j := 42, 2701

	p := &i         // 指向 i
	fmt.Println(*p) // 通过指针读取 i 的值
	*p = 21         // 通过指针设置 i 的值
	fmt.Println(i)  // 查看 i 的值

	p = &j         // 指向 j
	*p = *p / 37   // 通过指针对 j 进行除法运算
	fmt.Println(j) // 查看 j 的值
}
```

可看作**“间接引用”或“重定向”**

> Java引用赋新值是指向了另一个地方，地址值改变
>
> Go的指针赋新值是指针所指的地址处值的改变，指针指向的地址值没有变**（p没有变，*p改变）**

### 结构体

一组字段（field）

```go
package main

import "fmt"

type Vertex struct {
	X int
	Y int
}

func main() {
	fmt.Println(Vertex{1, 2})
}

```

> 如果我们有一个指向结构体的指针 `p`，那么可以通过 `(*p).X` 来访问其字段 `X`。不过这么写太啰嗦了，所以语言也允许我们使用隐式间接引用，直接写 `p.X` 就可以

**赋值**

```go
package main

import "fmt"

type Vertex struct {
	X, Y int
}

var (
	v1 = Vertex{1, 2}  // 创建一个 Vertex 类型的结构体
	v2 = Vertex{X: 1}  // Y:0 被隐式地赋予
	v3 = Vertex{}      // X:0 Y:0
	p  = &Vertex{1, 2} // 创建一个 *Vertex 类型的结构体（指针）
)

func main() {
	fmt.Println(v1, p, v2, v3)
}

```

### 数组和切片

#### 数组

```go
package main

import "fmt"

func main() {
	var a [2]string
	a[0] = "Hello"
	a[1] = "World"
	fmt.Println(a[0], a[1])
	fmt.Println(a)

	primes := [6]int{2, 3, 5, 7, 11, 13}
	fmt.Println(primes)
}
```

#### 切片

```go
package main

import "fmt"

func main() {
	primes := [6]int{2, 3, 5, 7, 11, 13}

	var s []int = primes[1:4]
	fmt.Println(s)
}
```

> 就像数组这一段的引用，并不实际存储这一段为另一个数组

- 切片文法类似于没有长度的数组文法
  - 这是一个数组文法：

    ```go
    [3]bool{true, true, false}
    ```

  - 下面这样则会创建一个和上面相同的数组，然后**构建一个引用了它的切片**：

    ```go
    []bool{true, true, false}
    ```

切片的长度和容量可通过表达式 `len(s)` 和 `cap(s)` 来获取，len就是切片长度，cap是从第一个开始数，数到数组末尾*（从0开始的切片，cap都为数组长度）*

切片的零值是 `nil`，nil 切片的长度和容量为 0 且没有底层数组

**make**

切片可以用内建函数 `make` 来创建，也是创建动态数组的方式

`make` 函数会分配一个元素为零值的数组并返回一个引用了它的切片：

```
a := make([]int, 5)  // len(a)=5
```

要指定它的容量，需向 `make` 传入第三个参数：

```
b := make([]int, 0, 5) // len(b)=0, cap(b)=5

b = b[:cap(b)] // len(b)=5, cap(b)=5
b = b[1:]      // len(b)=4, cap(b)=4
```

**append**

向切片追加元素，底层数组大小不够时会扩容

**range**

`for` 循环的 `range` 形式可遍历切片或映射

当使用 `for` 循环遍历切片时，每次迭代都会返回两个值。第一个值为当前元素的下标，第二个值为该下标所对应元素的一份副本*（副本存放在同一个地址空间）*

可以将下标或值赋予 `_` 来忽略它

```go
for i, _ := range pow
for _, value := range pow
```

若你只需要索引，忽略第二个变量即可

```go
for i := range pow
```

#### 总结

数组和切片：数组是固定容量的，切片不是，切片是动态的可扩容的，本质上是一个struct

### 两个实验

- 参数传递实验

  > ```go
  > package main
  > 
  > import "fmt"
  > 
  > func main() {
  > 	a1 := [3]int{1, 2, 3}
  > 	fmt.Println("-----before change arr-----")
  > 	fmt.Printf("数组原地址: %p\n", &a1)
  > 	fmt.Println("a1: ", a1)
  > 	fmt.Println("-----after change arr-----")
  > 	changeArray(a1)
  > 	fmt.Println("a1: ", a1)
  > 	
  > 	fmt.Println("**************************\n")
  > 	
  > 	s1 := a1[:]
  > 	fmt.Println("-----before change slice-----")
  > 	fmt.Printf("切片原地址: %p\n", &s1)
  > 	fmt.Println("s1: ", s1)
  > 	fmt.Println("-----after change slice-----")
  > 	changeSlice(s1)
  > 	fmt.Println("s1: ", s1)
  > 	fmt.Println("a1: ", a1)
  > 	
  > 	fmt.Println("**************************\n")
  > 	
  > 	fmt.Println("-----before append slice-----")
  > 	fmt.Printf("切片原地址: %p\n", &s1)
  > 	fmt.Println("s1: ", s1)
  > 	fmt.Println("-----after append slice-----")
  > 	appendAndChangeSlice(s1)
  > 	fmt.Println("s1: ", s1)
  > 	fmt.Println("a1: ", a1)
  > 
  > }
  > 
  > func changeArray(arr [3]int){
  > 	arr[0] = 999
  > 	fmt.Printf("数组参数地址: %p\n", &arr)
  > }
  > 
  > func changeSlice(slice []int){
  > 	slice[0] = 999
  > 	fmt.Printf("切片参数地址: %p\n", &slice)
  > 	fmt.Printf("切片首地址: %p\n", &slice[0])
  > }
  > 
  > func appendAndChangeSlice(slice []int){
  > 	slice = append(slice, 4, 5, 6, 7, 8)
  > 	slice[1] = 9999
  > 	fmt.Printf("切片参数地址: %p\n", &slice)
  > 	fmt.Printf("切片首地址: %p\n", &slice[0])
  > }
  > ```
  >
  > 结果为：
  >
  > ```
  > -----before change arr-----
  > a1:  [1 2 3]
  > 数组原地址: 0xc000016018
  > -----after change arr-----
  > 数组参数地址: 0xc000016048
  > a1:  [1 2 3]
  > **************************
  > 
  > -----before change slice-----
  > s1:  [1 2 3]
  > 切片原地址: 0xc00000c030
  > -----after change slice-----
  > 切片参数地址: 0xc00000c060
  > 切片首地址: 0xc000016018
  > s1:  [999 2 3]
  > a1:  [999 2 3]
  > **************************
  > 
  > -----before append slice-----
  > s1:  [999 2 3]
  > 切片原地址: 0xc00000c030
  > -----after append slice-----
  > 切片参数地址: 0xc00000c0a8
  > 切片首地址: 0xc000104040
  > s1:  [999 2 3]
  > a1:  [999 2 3]
  > ```

- 赋值传递实验

  > ```go
  > package main
  > 
  > import "fmt"
  > 
  > func main() {
  > 	a1 := [3]int{1, 2, 3}
  > 	fmt.Println("a1: ", a1)
  > 	fmt.Printf("a1地址: %p\n", &a1)
  > 	a2 := a1
  > 	fmt.Println("a2: ", a2)
  > 	fmt.Printf("a2地址: %p\n", &a2)
  > 	a2[0] = 999
  > 	fmt.Println("修改a2后, a2: ", a2, "  a1: ", a1)
  > 	
  > 	fmt.Println("**************************\n")
  > 	
  > 	s1 := a1[:]
  > 	fmt.Println("s1: ", s1)
  > 	fmt.Printf("s1地址: %p\n", &s1)
  > 	s2 := s1
  > 	fmt.Println("s2: ", s2)
  > 	fmt.Printf("s2地址: %p\n", &s2)
  > 	fmt.Printf("s1首地址: %p   s2首地址: %p\n", &s1[0], &s2[0])
  > 	
  > }
  > ```
  >
  > 结果为：
  >
  > ```
  > a1:  [1 2 3]
  > a1地址: 0xc0000be000
  > a2:  [1 2 3]
  > a2地址: 0xc0000be030
  > 修改a2后, a2:  [999 2 3]   a1:  [1 2 3]
  > **************************
  > 
  > s1:  [1 2 3]
  > s1地址: 0xc0000b4018
  > s2:  [1 2 3]
  > s2地址: 0xc0000b4048
  > s1首地址: 0xc0000be000   s2首地址: 0xc0000be000
  > ```

总结：

- 数组传参/赋值时传的是数组真实数据的拷贝，而不是地址拷贝（与Java不同，Java的数组是引用类型，这里是值类型）
- 切片传参/赋值时传的是切片的拷贝，生成新的切片，但指向的底层数组一致，也就是说，是浅拷贝（切片是引用类型）
- append后会新生成一个底层数组并令切片的Pointer指向它
- 切片首元素的地址==数组首元素的地址==数组地址

**在go函数中参数的传递可以是传值（对象的复制，需要开辟新的空间来存储该新对象）和传引用（指针的复制，和原来的指针指向同一个对象，但两个指针本身的内存地址不一致，只是指向的区域一致），建议使用指针，原因有两个：能够改变参数的值，避免大对象的复制操作节省内存** *（上述实验实际上都是值传递，都有复制过程）*

### go只有值传递

基于上面的实验得出结论：**go只有值传递**

首先明确值传递的定义：传递时会将数据复制一份；而引用传递：可以直接传递指针

区别在哪呢？假如有一个函数func foo(ptr *int)，我们有下面的函数：

```go
func main() {
    i := 123
    ptr := &i
	foo(ptr)
    fmt.Println(*ptr)
}

func foo(ptr *int) {
    i := 234
    ptr = &i
}
```

如果是引用传递，这里直接传指针，则会打印234，实际上可以尝试一下，打印的是123

说明传递的是指针本身的一个拷贝，也就是，**函数内外指针本身存放在不同的内存地址，但它们指向的内存地址是一致的**，这时去修改函数内的指针指向的内存地址，自然影响不到函数外的指针，所以是值传递

> - 浅拷贝的时候，指针本身也会被拷贝，区分开指针本身地址和它的指向地址
> - Java中同理，只有值传递，引用传递的时候会生成一个引用的副本，但指向地址一致

### 映射（Map）

在映射 `m` 中插入或修改元素：

```go
m[key] = elem
```

获取元素：

```go
elem = m[key]
```

删除元素：

```go
delete(m, key)
```

通过双赋值检测某个键是否存在：

```go
elem, ok = m[key]
```

若 `key` 在 `m` 中，`ok` 为 `true` ；否则，`ok` 为 `false`

若 `key` 不在映射中，那么 `elem` 是该映射元素类型的零值

同样的，当从映射中读取某个不存在的键时，结果是映射的元素类型的零值

**注** ：若 `elem` 或 `ok` 还未声明，可以使用短变量声明：

```
elem, ok := m[key]
```

## 函数

### 函数值

**函数也是值**。它们可以像其它值一样传递

函数值可以用作函数的参数或返回值

```go
package main

import (
	"fmt"
	"math"
)

func compute(fn func(float64, float64) float64) float64 {
	return fn(3, 4)
}
func add(x, y float64) float64 {
	return x + y
}
func main() {
	hypot := func(x, y float64) float64 {
		return math.Sqrt(x*x + y*y)
	}
	fmt.Println(hypot(5, 12))

	fmt.Println(compute(hypot))
	fmt.Println(compute(add))
}
```

### 闭包

> [https://www.sulinehk.com/post/golang-closure-details/](https://www.sulinehk.com/post/golang-closure-details/)

- 闭包是一个函数值，它引用了其函数体之外的变量*（除自身局部变量之外的变量）*。该函数可以访问并赋予其引用的变量的值，换句话说，该函数被这些变量“绑定”在一起，被引用的变量将和这个函数一同存在
- 闭包是函数和相关引用环境组成的实体

```go
package main

import "fmt"

func adder() func(int) int {
	sum := 0
    /* 下面的函数，即adder()的返回值，为一个闭包；因为它引用了自由变量sum，与sum绑定，且能通过函数值作为一个实体存在
     */
	return func(x int) int {
		sum += x
		return sum
	}
}

func main() {
    // 两个闭包实体：pos、neg
	pos, neg := adder(), adder()
	for i := 0; i < 10; i++ {
		fmt.Println(
			pos(i),
			neg(-2*i),
		)
	}
}
```

> 类比Java中的内部类对象：引用某外部变量；与外部变量绑定；作为方法和引用环境组成的实体存在*（Java的方法不能作为类似函数值的东西来保存，因此可以采用匿名内部类的方式，如下，方法为add，看作num和add绑定）*
>
> ```java
> public class Main{
>     public static void main(String[] args){
>         Outer o = new Outer();
>         // in是一个闭包
>         Inner in = o.inner();
>         in.add();
>         in.add();
>         in.add();
>         System.out.println(o.num); // 3 
>         //
>         o = null;
>         System.out.println(o);
>         System.out.println(in);  // o==null, in!=null, 内存泄漏
>     }
> }
> interface Inner{
>     void add();
> }
> class Outer{
>     public int num;
>     public Inner inner(){
>         return new Inner() {
>             @Override
>             public void add() {
>                 num++;
>             }
>         };
>     }
> }
> ```

## 方法

Go 没有类。不过你可以为结构体类型定义方法

方法就是一类带特殊的 **接收者** 参数的函数

> 方法接收者在它自己的参数列表内，位于 `func` 关键字和方法名之间
>
> 在此例中，`Abs` 方法拥有一个名为 `v`，类型为 `Vertex` 的接收者
>
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"math"
> )
> 
> type Vertex struct {
> 	X, Y float64
> }
> 
> func (v Vertex) Abs() float64 {
> 	return math.Sqrt(v.X*v.X + v.Y*v.Y)
> }
> 
> func main() {
> 	v := Vertex{3, 4}
> 	fmt.Println(v.Abs())
> }
> ```

接收者的类型定义和方法声明必须在同一包内；不能为内建类型声明方法

> 非结构体
>
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"math"
> )
> 
> type MyFloat float64
> 
> func (f MyFloat) Abs() float64 {
> 	if f < 0 {
> 		return float64(-f)
> 	}
> 	return float64(f)
> }
> 
> func main() {
> 	f := MyFloat(-math.Sqrt2)
> 	fmt.Println(f.Abs())
> }
> ```

### 指针接收者

```go
package main

import (
	"fmt"
	"math"
)

type Vertex struct {
	X, Y float64
}

func (v Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

// 
func (v *Vertex) Scale(f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

func main() {
	v := Vertex{3, 4}
	v.Scale(10)
	fmt.Println(v.Abs())  // 50
}
```

注意，如果将此处的*Vertex改成Vertex，结果为5，因为go的值传递和Java不一样，会新生成一个浅拷贝对象；引用传递则是拷贝地址

### 方法即函数&隐式的处理

- 方法即函数：方法都可以写成函数的形式

  ```go
  package main
  
  import (
  	"fmt"
  	"math"
  )
  
  type Vertex struct {
  	X, Y float64
  }
  
  func Abs(v Vertex) float64 {
  	return math.Sqrt(v.X*v.X + v.Y*v.Y)
  }
  
  func Scale(v *Vertex, f float64) {
  	v.X = v.X * f
  	v.Y = v.Y * f
  }
  
  func main() {
  	v := Vertex{3, 4}
  	Scale(&v, 10)
  	fmt.Println(Abs(v))
  }
  ```

  这里的Scale方法的Vertex必须写成*Vertex，否则就是值传递了，创建一个Vertex副本

  但在方法的写法中却不用，为什么？

- 隐式的处理：在方法的写法中，会隐式转换成指针的形式

  比较前两个程序，你大概会注意到带指针参数的函数必须接受一个指针：

  ```go
  var v Vertex
  ScaleFunc(v, 5)  // 编译错误！
  ScaleFunc(&v, 5) // OK
  ```

  而以指针为接收者的方法被调用时，接收者既能为值又能为指针：

  ```go
  var v Vertex
  v.Scale(5)  // OK
  p := &v
  p.Scale(10) // OK
  ```

  对于语句 `v.Scale(5)`，即便 `v` 是个值而非指针，带指针接收者的方法也能被直接调用。 也就是说，由于 `Scale` 方法有一个指针接收者，为方便起见，Go 会将语句 `v.Scale(5)` 编译为 `(&v).Scale(5)`

## 接口

```go
package main

import (
	"fmt"
	"math"
)

type Abser interface {
	Abs() float64
}

func main() {
	var a Abser
	f := MyFloat(-math.Sqrt2)
	v := Vertex{3, 4}

	a = f  // a MyFloat 实现了 Abser
	a = &v // a *Vertex 实现了 Abser



	fmt.Println(a.Abs())
}

type MyFloat float64

func (f MyFloat) Abs() float64 {
	if f < 0 {
		return float64(-f)
	}
	return float64(f)
}

type Vertex struct {
	X, Y float64
}

func (v *Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}
```

类型通过实现一个接口的所有方法来实现该接口。既然无需专门显式声明，也就没有“implements”关键字

隐式接口从接口的实现中解耦了定义，这样接口的实现可以出现在任何包中，无需提前准备

因此，也就无需在每一个实现上增加新的接口名称，这样同时也鼓励了明确的接口定义

```go
package main

import "fmt"

type I interface {
	M()
}

type T struct {
	S string
}

// 此方法表示类型 T 实现了接口 I，但我们无需显式声明此事。
func (t T) M() {
	fmt.Println(t.S)
}

func main() {
	var i I = T{"hello"}
	i.M()
}
```

### 接口值

接口也是值。它们可以像其它值一样传递；接口值可以用作函数的参数或返回值。

在内部，接口值可以看做包含值和具体类型的元组：`(value, type)`

接口值保存了一个具体底层类型的具体值；接口值调用方法时会执行其底层类型的同名方法。见describe方法：

```go
package main

import (
	"fmt"
	"math"
)

type I interface {
	M()
}

type T struct {
	S string
}

func (t *T) M() {
	fmt.Println(t.S)
}

type F float64

func (f F) M() {
	fmt.Println(f)
}

func main() {
	var i I

	i = &T{"Hello"}
	describe(i)
	i.M()

	i = F(math.Pi)
	describe(i)
	i.M()
}

func describe(i I) {
	fmt.Printf("(%v, %T)\n", i, i)
}
```

### 空接口

指定了零个方法的接口值被称为 *空接口：*

```go
interface{}
```

空接口可保存任何类型的值。（因为每个类型都至少实现了零个方法）

空接口被用来处理未知类型的值。例如，`fmt.Print` 可接受类型为 `interface{}` 的任意数量的参数

### 类型断言

断言接口变量是否为某个类型，有点像instanceof

```go
t := i.(T)
```

该语句断言接口值 `i` 保存了具体类型 `T`，并将其底层类型为 `T` 的值赋予变量 `t`

若 `i` 并未保存 `T` 类型的值，该语句就会触发一个恐慌（panic）

为了 **判断** 一个接口值是否保存了一个特定的类型，类型断言可返回两个值：其底层值以及一个报告断言是否成功的布尔值

```go
t, ok := i.(T)
```

若 `i` 保存了一个 `T`，那么 `t` 将会是其底层值，而 `ok` 为 `true`

否则，`ok` 将为 `false` 而 `t` 将为 `T` 类型的零值，程序并不会产生恐慌

> 请注意这种语法和读取一个映射时的相同之处

### 类型选择

```go
switch v := i.(type) {
case T:
    // v 的类型为 T
case S:
    // v 的类型为 S
default:
    // 没有匹配，v 与 i 的类型相同
}
```

### Stringer

> 类似Java重写toString

`fmt`包中定义的 `Stringer`是最普遍的接口之一

```go
type Stringer interface {
    String() string
}
```

`Stringer` 是一个可以用字符串描述自己的类型。`fmt` 包（还有很多包）都通过此接口来打印值

### 错误

Go 程序使用 `error` 值来表示错误状态

与 `fmt.Stringer` 类似，`error` 类型是一个内建接口：

```go
type error interface {
    Error() string
}
```

（与 `fmt.Stringer` 类似，`fmt` 包在打印值时也会满足 `error`）

通常函数会返回一个 `error` 值，调用的它的代码应当判断这个错误是否等于 `nil` 来进行错误处理。

```go
i, err := strconv.Atoi("42")
if err != nil {
    fmt.Printf("couldn't convert number: %v\n", err)
    return
}
fmt.Println("Converted integer:", i)
```

例子：

```go
package main

import (
	"fmt"
	"time"
)

type MyError struct {
	When time.Time
	What string
}

func (e *MyError) Error() string {
	return fmt.Sprintf("at %v, %s",
		e.When, e.What)
}

func run() error {
	return &MyError{
		time.Now(),
		"it didn't work",
	}
}

func main() {
	if err := run(); err != nil {
		fmt.Println(err)
	}
}
```

### Reader

`io` 包指定了 `io.Reader` 接口，它表示从数据流的末尾进行读取

Go 标准库包含了该接口的许多实现，包括文件、网络连接、压缩和加密等等

`io.Reader` 接口有一个 `Read` 方法：

```go
func (T) Read(b []byte) (n int, err error)
```

`Read` 用数据填充给定的字节切片并返回填充的字节数和错误值。在遇到数据流的结尾时，它会返回一个 `io.EOF` 错误

示例代码创建了一个 strings.Reader 并以每次 8 字节的速度读取它的输出

```go
package main

import (
	"fmt"
	"io"
	"strings"
)

func main() {
	r := strings.NewReader("Hello, Reader!")

	b := make([]byte, 8)
	for {
		n, err := r.Read(b)
		fmt.Printf("n = %v err = %v b = %v\n", n, err, b)
		fmt.Printf("b[:n] = %q\n", b[:n])
		if err == io.EOF {
			break
		}
	}
}
```

# goroutine

## goroutine

轻量级线程、协程、用户级线程

```go
go f(x, y, z)
```

会启动一个新的 Go 程并执行

```go
f(x, y, z)
```

`f`, `x`, `y` 和 `z` 的求值发生在当前的 goroutine中，而 `f` 的执行发生在新的 Go 程中

Go 程在相同的地址空间中运行，因此在访问共享的内存时必须进行同步。sync 包提供了这种能力，不过在 Go 中并不经常用到，因为还有其它的办法

## 信道

信道是带有类型的管道，你可以通过它用信道操作符 `<-` 来发送或者接收值

```go
ch <- v    // 将 v 发送至信道 ch
v := <-ch  // 从 ch 接收值并赋予 v
```

> 箭头就是数据流的方向

和映射与切片一样，信道在使用前必须创建：

```go
ch := make(chan int)
// ch := make(chan int, 2) // 带缓冲的
```

默认情况下，发送和接收操作在另一端准备好之前都会**阻塞**。这使得 goroutine 可以在没有显式的锁或竞态变量的情况下进行同步

```go
package main

import "fmt"

func sum(s []int, c chan int) {
	sum := 0
	for _, v := range s {
		sum += v
	}
	c <- sum // 将和送入 c
}

func main() {
	s := []int{7, 2, 8, -9, 4, 0}

	c := make(chan int)
	go sum(s[:len(s)/2], c)
	go sum(s[len(s)/2:], c)
	x, y := <-c, <-c // 从 c 中接收

	fmt.Println(x, y, x+y)
}
```

> 主线程的阻塞会视为死锁

> **为什么卡住不显示？记得问**
>
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	go func(){
> 		ch := make(chan int, 2)
> 		ch <- 1
> 		ch <- 2
> 		fmt.Println(<-ch)
> 		fmt.Println(<-ch)
> 	}()
> }
> ```

### range 和 close

发送者可通过 `close` 关闭一个信道来表示没有需要发送的值了。**接收者**可以通过为接收表达式分配第二个参数来测试信道是否被关闭：若没有值可以接收且信道已被关闭，那么在执行完

```
v, ok := <-ch
```

之后 `ok` 会被设置为 `false`

循环 `for i := range c` 会不断从信道接收值，直到它被关闭

- 只有发送者才能关闭信道，而接收者不能。向一个已经关闭的信道发送数据会引发程序恐慌（panic）
- 信道与文件不同，通常情况下无需关闭它们。只有在必须告诉接收者不再有需要发送的值时才有必要关闭，例如终止一个 `range` 循环

### select

`select` 语句使一个 Go 程可以等待多个通信操作

`select` 会阻塞到某个分支可以继续执行为止，这时就会执行该分支。当多个分支都准备好时会随机选择一个执行

```go
package main

import "fmt"

func fibonacci(c, quit chan int) {
	x, y := 0, 1
	for {
		select {
		case c <- x:
			x, y = y, x+y
		case <-quit:
			fmt.Println("quit")
			return
		}
	}
}

func main() {
	c := make(chan int)
	quit := make(chan int)
	go func() {
		for i := 0; i < 10; i++ {
			fmt.Println(<-c)
		}
		quit <- 0
	}()
	fibonacci(c, quit)
}
```

当 `select` 中的其它分支都没有准备好时，`default` 分支就会执行

为了在尝试发送或者接收时不发生阻塞，可使用 `default` 分支：

```go
select {
case i := <-c:
    // 使用 i
default:
    // 从 c 中接收会阻塞时执行
}
```

## sync.Mutex

我们已经看到信道非常适合在各个 goroutine 间进行通信

但是如果我们并不需要通信呢？比如说，若我们只是想保证每次只有一个 goroutine 能够访问一个共享的变量，从而避免冲突？

Go 标准库中提供了 sync.Mutex 互斥锁类型及其两个方法：

- `Lock`
- `Unlock`

我们可以通过在代码前调用 `Lock` 方法，在代码后调用 `Unlock` 方法来保证一段代码的互斥执行。参见 `Inc` 方法

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// SafeCounter 的并发使用是安全的。
type SafeCounter struct {
	v   map[string]int
	mux sync.Mutex
}

// Inc 增加给定 key 的计数器的值。
func (c *SafeCounter) Inc(key string) {
	c.mux.Lock()
	// Lock 之后同一时刻只有一个 goroutine 能访问 c.v
	c.v[key]++
	c.mux.Unlock()
}

// Value 返回给定 key 的计数器的当前值。
func (c *SafeCounter) Value(key string) int {
	c.mux.Lock()
	// Lock 之后同一时刻只有一个 goroutine 能访问 c.v
	defer c.mux.Unlock()
	return c.v[key]
}

func main() {
	c := SafeCounter{v: make(map[string]int)}
	for i := 0; i < 1000; i++ {
		go c.Inc("somekey")
	}

	time.Sleep(time.Second)
	fmt.Println(c.Value("somekey"))
}
```

## sync.WaitGroup

sync.WaitGroup：等待子协程结束

- Add：++
- Wait：Wait Until all end
- Done：--

**注意！**：在开辟子协程之前就Add，如果在子协程内Add，即Add和Done都在子协程内，可能在Add之前主协程就会判断到等待协程数为0，这个子协程就失效了

> [https://tour.go-zh.org/concurrency/10](https://tour.go-zh.org/concurrency/10)

# goWeb

## net包

简单的使用：`resp, err := http.Get(url)`，resp为返回的响应

```go
// Fetch prints the content found at a URL.
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"os"
)

func main() {
	for _, url := range os.Args[1:] {
		resp, err := http.Get(url)
		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
			os.Exit(1)
		}
		b, err := ioutil.ReadAll(resp.Body)
		resp.Body.Close()
		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch: reading %s: %v\n", url, err)
			os.Exit(1)
		}
		fmt.Printf("%s", b)
	}
}
```

- resp.Body：响应体
- resp.Status：状态码

## 简单web服务

Go语言的内置库使得写一个web服务器变得异常简单

```go
// Server1 is a minimal "echo" server.
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", handler) // each request calls handler
	log.Fatal(http.ListenAndServe("localhost:8000", nil))
}

// handler echoes the Path component of the request URL r.
func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}
```

- main函数将所有发送到 **/** 路径下的请求和handler函数关联起来，**/** 开头的请求其实就是所有发送到当前站点上的请求，服务监听8000端口
- 发送到这个服务的“请求”是一个http.Request类型的对象，这个对象中包含了请求中的一系列相关字段，其中就包括我们需要的URL
- 当请求到达服务器时，这个请求会被传给handler函数来处理，这个函数会将/hello这个路径从请求的URL中解析出来，然后把其发送到响应中，这里我们用的是标准输出流的fmt.Fprintf
- 服务器每一次接收请求处理时都会另起一个goroutine，这样服务器就可以同一时间处理多个请求

> Linux/Mac OS下，go run ...的末尾加&进行后台运行

[取参数](https://blog.csdn.net/kenkao/article/details/47857757)