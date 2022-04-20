# [返回](/)

# go语法

## 基本语法

编译型、强类型（隐式类型）、动态类型

- 非驼峰命名，首字母大写

- 类型声明在后

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

- 数组传参/赋值时传的是数组真实数据的拷贝，而不是地址拷贝（与Java不同）
- 切片传参/赋值时传的是切片真实数据的拷贝，生成新的切片，但指向的底层数组一致，也就是说，是浅拷贝
- append后会新生成一个底层数组并令切片的Pointer指向它
- 切片首元素的地址==数组首元素的地址==数组地址

**在go函数中参数的传递可以是传值（对象的复制，需要开辟新的空间来存储该新对象）和传引用（指针的复制，和原来的指针指向同一个对象），建议使用指针，原因有两个：能够改变参数的值，避免大对象的复制操作节省内存** *（上述实验实际上都是传值）*

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