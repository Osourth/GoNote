### golang使用总结

#### 一. 语法

##### 1. 类型断言

###### 1.1 简介

> 写法为value, ok := em.(T)   如果确保em 是同类型的时候可以直接使用value:=em.(T)一般用于switch语句中
>
> 下面将会讲解到:
>
> ​	em代表要判断的变量 
> ​	T代表被判断的类型
> ​	value代表返回的值
> ​	ok代表是否为该类型

###### 1.2 注意事项

> * **em必须为initerface类型才可以进行类型断言**
>
>   以下代码会报错：
>
>   ```go
>   	s := "BrainWu"
>   	if v, ok := s.(string); ok {
>   		fmt.Println(v)
>   	}
>   ```
>
>   报错信息：
>
>   ```go
>   invalid type assertion: s.(string) (non-interface type string on left)
>   ```
>
>   在这里只要是在声明时或函数传进来的参数不是interface类型那么做类型断言都是回报 non-interface的错误的
>
>   所以我们只能通过将s作为一个interface{}的方法来进行类型断言 如下代码所示:
>
>   ```go
>   	s := "BrainWu"
>   	if v, ok := interface{}(s).(string); ok {
>   		fmt.Println(v)
>   	}
>   ```
>

##### 2. type alias

> type alias这个特性的主要目的是用于已经定义的类型，在package之间的移动时的兼容。比如我们有一个导出的类型`flysnow.org/lib/T1`，现在要迁移到另外一个package中, 比如`flysnow.org/lib2/T1`中。
>
> 没有type alias的时候我们这么做，就会导致其他第三方引用旧的package路径的代码，都要统一修改，不然无法使用。
>
> 有了type alias就不一样了，类型T1的实现我们可以迁移到lib2下，同时我们在原来的lib下定义一个lib2下T1的别名，这样第三方的引用就可以不用修改，也可以正常使用，只需要兼容一段时间，再彻底的去掉旧的package里的类型兼容，这样就可以渐进式的重构我们的代码，而不是一刀切。
>
> ```go
> type T1=lib2.T1
> ```

###### 2.1 type alias vs defintion

> 我们基于一个类型创建一个新类型，称之为defintion；基于一个类型创建一个别名，称之为alias，这就是他们最大的区别。
>
> ```
> type MyInt1 int		//defintion
> type MyInt2 = int	//type alias
> ```
>
> 第一行代码是基于基本类型int创建了新类型MyInt1，第二行是创建的一个int的类型别名MyInt2，注意类型别名的定义是`=`。
>
> ```go
> var i int =0
> var i1 MyInt1 = i //error
> var i2 MyInt2 = i
> fmt.Println(i1,i2)
> ```
>
> 仔细看这个示例，第二行把一个`int`类型的变量`i`,赋值给`MyInt1`类型的变量`i1`会被提示编译错误：类型无法转换。但是第三行把`int`类型的变量`i`,赋值给`MyInt2`类型的变量`i2`就可以，不会提示错误。
>
> 从这个例子也可以看出来，这两种定义方式的不同，因为Go是强类型语言，所以类型之间的转换必须强制转换，因为`int`和`MyInt1`是不同的类型，所以这里会报编译错误。
>
> 但是因为`MyInt2`只是`int`的一个别名，本质上还是一个`int`类型，所以可以直接赋值，不会有问题。

###### 2.2 类型实现

> 每个类型都可以通过接受者的方式，添加属于它自己的方法，我们看下通过type alias的类型是否可以，以及拥有哪些方法。
>
> ```go
> type MyInt1 int
> type MyInt2 = int
> 
> func (i MyInt1) m1(){
> 	fmt.Println("MyInt1.m1")
> }
> 
> func (i MyInt2) m2(){
> 	fmt.Println("MyInt2.m2")
> }
> 
> func main() {
> 	var i1 MyInt1
> 	var i2 MyInt2
> 	i1.m1()
> 	i2.m2()
> }
> ```
>
> 以上示例代码看着是没有任何问题，但是我们编译的时候会提示：
>
> ```go
> i2.m2 undefined (type int has no field or method m2)
> cannot define new methods on non-local type int
> ```
>
> 这里面有2个错误，一个是提示类型`int`没有`m2`这个方法，所以我们不能调用，因为`MyInt2`本质上就是`int`。
>
> 第二个错误是我们不能为`int`类型添加新方法，什么意思呢？因为`int`是一个非本地类型，所以我们不能为其增加方法。既然这样，那我们自定义个struct类型试试。
>
> ```go
> type User struct {
> 
> }
> type MyUser1 User
> type MyUser2 = User
> 
> func (i MyUser1) m1(){
> 	fmt.Println("MyUser1.m1")
> }
> 
> func (i MyUser2) m2(){
> 	fmt.Println("MyUser2.m2")
> }
> 
> //Blog:www.flysnow.org
> //Wechat:flysnow_org
> func main() {
> 	var i1 MyUser1
> 	var i2 MyUser2
> 	i1.m1()
> 	i2.m2()
> }
> ```
>
> 换成struct，正常运行。所以本地定义的类型的别名，还是可以为其添加方法的。现在我们接着上面的例子，看一个有趣的现象，我在main函数里增加如下代码：
>
> ```go
> 	var i User
> 	i.m2()
> ```
>
> 然后运行，发现，可以正常运行。是不是很奇怪，我们并没有为类型`User` 定义方法啊，怎么可以调用呢？这就得益于type alias，`MyUser2`完全等价于`User`，所以为`MyUser2`定义方法，等于就为`User`定义了方法，反之，亦然。
>
> 但是对于新定义的类型`MyUser1`就不行了，因为它完全是个新类型，所以`User`的方法，`MyUser`是没有的。这里不再举例，大家自己可以试试。
>
> 还有一点需要注意，因为`MyUser2`完全等价于`User`，所以`User`已经有的方法，`MyUser2`不能再声明，反之亦然，如果定义了会有如下提示：
>
> ```go
> ./main.go:37:6: User.m1 redeclared in this block
> 	previous declaration at ./main.go:31:6
> ```
>
> 其实就是重复声明的意思，不能再次重复声明了。

###### 2.3 接口实现

> 上面的小结我们可以发现，`User`和`MyUser2`是等价的，并且其中一个新增了方法，另外一个也会有。那么基于此推导出，一个实现了某个接口，另外一个也会实现。现在验证一下：
>
> ```go
> type I interface {
> 	m2()
> }
> 
> type User struct {
> 
> }
> 
> type MyUser2 = User
> 
> func (i User) m(){
> 	fmt.Println("User.m")
> }
> 
> func (i MyUser2) m2(){
> 	fmt.Println("MyUser2.m2")
> }
> 
> func main() {
> 	var u User
> 	var u2 MyUser2
> 	var i1 I =u
> 	var i2 I =u2
> 
> 	fmt.Println(i1,i2)
> }
> ```
>
> 定义了一个接口`I`，从代码上看，只有`MyUser2`实现了它，但是我们代码的演示中，发现`User`也实现了接口`I`，所以这就验证了我们的推到是正确的，返回来如果`User`实现了某个接口，那么它的type alias也同样会实现这个接口。
>
> 以上讲了很多示例都是类型struct的别名，我们看下接口interface的type alias是否也是等价的。
>
> ```go
> type I interface {
> 	m()
> }
> 
> type MyI1 I
> type MyI2 = I
> 
> type MyInt int
> func (i MyInt) m(){
> 	fmt.Println("MyInt.m")
> }
> ```
>
> 定义了一个接口`I`，`MyI1`是基于`I`的新类型；`MyI2`是`I`的类型别名；`MyInt`实现了接口`I`。下面进行测试。
>
> ```go
> func main() {
> 	//赋值实现类型MyInt
> 	var i I = MyInt(0)
> 	var i1 MyI1 = MyInt(0)
> 	var i2 MyI2 = MyInt(0)
> 
> 	//接口间相互赋值
> 	i = i1
> 	i = i2
> 	i1 = i2
> 	i1 = i
> 	i2 = i
> 	i2 = i1
> }
> ```
>
> 以上代码运行是正常的，这个是前面讲的具体类型（struct，int等）的type alias不一样，只要实现了接口，就可以相互赋值，管你是新定义的接口`MyI1`，还是接口的别名`MyI2`。

###### 2.4 类型的嵌套

> 我们都知道type alias的两个类型是等价的，但是他们在类型嵌套时有些不一样。
>
> ```go
> func main() {
> 	my:=MyStruct{}
> 	my.T2.m1()
> }
> 
> type T1 struct {
> 
> }
> 
> func (t T1) m1(){
> 	fmt.Println("T1.m1")
> }
> 
> type T2 = T1
> 
> type MyStruct struct {
> 	T2
> }
> ```
>
> 示例中`T2`是`T1`的别名，但是我们把T2嵌套在`MyStruct`中，在调用的时候只能通过`T2`这个名称调用，而不能通过`T1`，会提示没这个字段的。反过来也一样。
>
> 这是因为`T1`,`T2`是两个名称，虽然他们等价，但他们是具有两个不同名字的等价类型，所以在类型嵌套的时候，就是两个字段。
>
> 当然我们可以把`T1`,`T2`同时嵌入到`MyStrut`中，进行分别调用。
>
> ```go
> func main() {
> 	my:=MyStruct{}
> 	my.T2.m1()
> 	my.T1.m1()
> }
> 
> type MyStruct struct {
> 	T2
> 	T1
> }
> ```
>
> 以上也是可以正常运行的，证明这是具有两个不同名字的，同种类型的字段。
>
> 下面我们做个有趣的实验，把`main`方法的代码改为如下：
>
> ```
> func main() {
> 	my:=MyStruct{}
> 	my.m1()
> }
> ```
>
> 猜猜是不是可以正常编译运行呢？答应可能出乎意料，是不能正常编译的，提示如下：
>
> ```go
> ./main.go:25:4: ambiguous selector my.m1
> ```
>
> 其实想想很简单，不知道该调用哪个，太模糊了，匹配不了，不然该用`T1`的`m1`,还是`T2`的`m1`。这种结果不限于方法，字段也也一样；也不限于type alias，type defintion也是一样的，只要有重复的方法、字段，就会有这种提示，因为不知道该选择哪个。

###### 2.5 类型循环

>  type alias的声明，一定要留意类型循环，不要产生了循环，一旦产生，就会编译不通过，那么什么是类型循环呢。假如`type T2 = T1`,那么`T1`绝对不能直接、或者间接的引用到`T2`，一旦有，就会类型循环。
>
> ```
> type T2 = *T2
> 
> type T2 = MyStruct
> type MyStruct struct {
> 	T1
> 	T2
> }
> ```
>
> 以上两种定义都是类型循环，我们自己在使用的过程中，要避免这种定义的出现。

###### 2.6 byte and rune

> 这两个类型一个是int8的别名，一个是int32的别名，在Go 1.9之前，他们是这么定义的。
>
> ```
> type byte byte
> 
> type rune rune
> ```
>
> 现在Go 1.9有了type alias这个新特性后，他们的定义就变成如下了：
>
> ```
> type byte = uint8
> 
> type rune = int32
> ```
>
> 恩，非常很省事和简洁。

###### 2.7 导出未导出的类型

> type alias还有一个功能，可以导出一个未被导出的类型。
>
> ```go
> package lib
> 
> type user struct {
> 	name string
> 	Email string
> }
> 
> func (u user) getName() string {
> 	return u.name
> }
> 
> func (u user) GetEmail() string {
> 	return u.Email
> }
> 
> //把这个user导出为User
> type User = user
> ```
>
> user`本身是一个未导出的类型，不能被其他package访问，但是我们可以通过`type User = user`，定义一个`User`，这样这个`User`就可以被其他package访问了，可以使用`user`类型导出的字段和方法，示例中是`Email`字段和`GetEmail`方法，另外未被导出`name`字段和`getName`方法是不能被其他package使用的。

##### 3. new 和 make

> Go语言中new和make是内建的两个函数，主要用来创建分配类型内存。在我们定义生成变量的时候，可能会觉得有点迷惑，其实他们的规则很简单，下面我们就通过一些示例说明他们的区别和使用。

###### 3.1 变量的声明

> ```go
> var i int
> var s string
> ```
>
> 变量的声明我们可以通过`var`关键字，然后就可以在程序中使用。当我们不指定变量的默认值时，这些变量的默认值是他们的零值，比如`int`类型的零值是0,`string`类型的零值是`""`，引用类型的零值是`nil`。
>
> 对于例子中的两种类型的声明，我们可以直接使用，对其进行赋值输出。但是如果我们换成**引用类型**呢？
>
> ```go
> package main
> 
> import (
> 	"fmt"
> )
> 
> func main() {
> 	var i *int
> 	*i=10
> 	fmt.Println(*i)
> 
> }
> ```
>
> 这个例子会打印出什么？`0`还是`10`?。以上全错，运行的时候会painc，原因如下
>
> ```go
> panic: runtime error: invalid memory address or nil pointer dereference
> ```
>
> 从这个提示中可以看出，**对于引用类型的变量，我们不光要声明它，还要为它分配内容空间**，否则我们的值放在哪里去呢？这就是上面错误提示的原因。
>
> 对于值类型的声明不需要，是因为已经默认帮我们分配好了。
>
> 要分配内存，就引出来今天的`new`和`make`。

###### 3.2 new

> 对于上面的问题我们如何解决呢？既然我们知道了没有为其分配内存，那么我们使用new分配一个吧。
>
> ```go
> func main() {
> 	var i *int
> 	i=new(int)
> 	*i=10
> 	fmt.Println(*i)
> 
> }
> ```
>
> 现在再运行程序，完美PASS，打印`10`。现在让我们看下`new`这个内置的函数。
>
> ```go
> // The new built-in function allocates memory. The first argument is a type,
> // not a value, and the value returned is a pointer to a newly
> // allocated zero value of that type.
> func new(Type) *Type
> ```
>
> 它只接受一个参数，这个参数是一个类型，分配好内存后，返回一个指向该类型内存地址的指针。同时请注意它同时把分配的内存置为零，也就是类型的零值。
>
> 我们的例子中，如果没有`*i=10`，那么打印的就是0。这里体现不出来`new`函数这种内存置为零的好处，我们再看一个例子。
>
> ```
> func main() {
> 	u:=new(user)
> 	u.lock.Lock()
> 	u.name = "张三"
> 	u.lock.Unlock()
> 
> 	fmt.Println(u)
> 
> }
> 
> type user struct {
> 	lock sync.Mutex
> 	name string
> 	age int
> }
> ```
>
> 示例中的`user`类型中的`lock`字段我不用初始化，直接可以拿来用，不会有无效内存引用异常，因为它已经被零值了。
>
> 这就是`new`，它返回的永远是类型的指针，指向分配类型的内存地址。

###### 3.3 make

> `make`也是用于内存分配的，但是和`new`不同，它只用于`chan`、`map`以及切片的内存创建，而且它返回的类型就是这三个类型本身，而不是他们的指针类型，因为这三种类型就是引用类型，所以就没有必要返回他们的指针了。
>
> 注意，因为这三种类型是引用类型，所以必须得初始化，但是不是置为零值，这个和`new`是不一样的。
>
> ```
> func make(t Type, size ...IntegerType) Type
> ```
>
> 从函数声明中可以看到，返回的还是该类型。

###### 3.4 总结

> * 两者异同
>
>   所以从这里可以看的很明白了，二者都是内存的分配（堆上），但是`make`只用于slice、map以及channel的初始化（非零值）；而`new`用于类型的内存分配，并且内存置为零。所以在我们编写程序的时候，就可以根据自己的需要很好的选择了。
>
>   `make`返回的还是这三个引用类型本身；而`new`返回的是指向类型的指针。
>
> * **new**不常用
>
>   所以有new这个内置函数，可以给我们分配一块内存让我们使用，但是现实的编码中，它是不常用的。我们通常都是采用短语句声明以及结构体的字面量达到我们的目的，比如：
>
>   ```go
>   i:=0
>   u:=user{}
>   ```
>
>   这样更简洁方便，而且不会涉及到指针这种比麻烦的操作。
>
>   `make`函数是无可替代的，我们在使用slice、map以及channel的时候，还是要使用`make`进行初始化，然后才才可以对他们进行操作。

##### 4.值类型和引用类型

> 对于了解一门语言来说，会关心我们在函数调用的时候，参数到底是传的值，还是引用？
>
> 其实对于传值和传引用，是一个比较古老的话题，做研发的都有这个概念，但是可能不是非常清楚。对于我们做Go语言开发的来说，也想知道到底是什么传递。
>
> 那么我们先来看看什么是值传递，什么是引用传递。

###### 4.1 值传递

> 传值的意思是：函数传递的总是原来这个东西的一个副本，一副拷贝。比如我们传递一个`int`类型的参数，传递的其实是这个参数的一个副本；传递一个指针类型的参数，其实传递的是这个该指针的一份拷贝，而不是这个指针指向的值。
>
> 对于int这类基础类型我们可以很好的理解，它们就是一个拷贝，但是指针呢？我们觉得可以通过它修改原来的值，怎么会是一个拷贝呢？下面我们看个例子。
>
> ```go
> func main() {
> 	i:=10
> 	ip:=&i
> 	fmt.Printf("原始指针的内存地址是：%p\n",&ip)
> 	modify(ip)
> 	fmt.Println("int值被修改了，新值为:",i)
> }
> 
>  func modify(ip *int){
> 	 fmt.Printf("函数里接收到的指针的内存地址是：%p\n",&ip)
>  	*ip=1
>  }
> ```
>
> 我们运行，可以看到输入结果如下：
>
> ```
> 原始指针的内存地址是：0xc42000c028
> 函数里接收到的指针的内存地址是：0xc42000c038
> int值被修改了，新值为: 1
> ```
>
> 首先我们要知道，任何存放在内存里的东西都有自己的地址，指针也不例外，它虽然指向别的数据，但是也有存放该指针的内存。
>
> 所以通过输出我们可以看到，这是一个指针的拷贝，因为存放这两个指针的内存地址是不同的，虽然指针的值相同，但是是两个不同的指针。
>
> ![指针传递解释](https://www.flysnow.org/uploads/2018/02/pointer-param.png)
>
> 通过上面的图，可以更好的理解。 首先我们看到，我们声明了一个变量`i`,值为`10`,它的内存存放地址是`0xc420018070`,通过这个内存地址，我们可以找到变量`i`,这个内存地址也就是变量`i`的指针`ip`。
>
> 指针`ip`也是一个指针类型的变量，它也需要内存存放它，它的内存地址是多少呢？是`0xc42000c028`。 在我们传递指针变量`ip`给`modify`函数的时候，是该指针变量的拷贝,所以新拷贝的指针变量`ip`，它的内存地址已经变了，是新的`0xc42000c038`。
>
> 不管是`0xc42000c028`还是`0xc42000c038`，我们都可以称之为指针的指针，他们指向同一个指针`0xc420018070`，这个`0xc420018070`又指向变量`i`,这也就是为什么我们可以修改变量`i`的值。

###### 4.2 引用传递

> Go语言(Golang)是没有引用传递的，这里我不能使用Go举例子，但是可以通过说明描述。
>
> 以上面的例子为例，如果在`modify`函数里打印出来的内存地址是不变的，也是`0xc42000c028`，那么就是引用传递。

###### 4.3 Map

> 了解清楚了传值和传引用，但是对于Map类型来说，可能觉得还是迷惑，一来我们可以通过方法修改它的内容，二来它没有明显的指针。
>
> ```go
> func main() {
> 	persons:=make(map[string]int)
> 	persons["张三"]=19
> 
> 	mp:=&persons
> 
> 	fmt.Printf("原始map的内存地址是：%p\n",mp)
> 	modify(persons)
> 	fmt.Println("map值被修改了，新值为:",persons)
> }
> 
>  func modify(p map[string]int){
> 	 fmt.Printf("函数里接收到map的内存地址是：%p\n",&p)
> 	 p["张三"]=20
>  }
> ```
>
> 运行打印输出：
>
> ```
> 原始map的内存地址是：0xc42000c028
> 函数里接收到map的内存地址是：0xc42000c038
> map值被修改了，新值为: map[张三:20]
> ```
>
> 两个内存地址是不一样的，所以这又是一个值传递（值的拷贝），那么为什么我们可以修改Map的内容呢？先不急，我们先看一个自己实现的`struct`。
>
> ```go
> func main() {
> 	p:=Person{"张三"}
> 	fmt.Printf("原始Person的内存地址是：%p\n",&p)
> 	modify(p)
> 	fmt.Println(p)
> }
> 
> type Person struct {
> 	Name string
> }
> 
>  func modify(p Person) {
> 	 fmt.Printf("函数里接收到Person的内存地址是：%p\n",&p)
> 	 p.Name = "李四"
>  }
> ```
>
> 运行打印输出：
>
> ```
> 原始Person的内存地址是：0xc4200721b0
> 函数里接收到Person的内存地址是：0xc4200721c0
> {张三}
> ```
>
> 我们发现，我们自己定义的`Person`类型，在函数传参的时候也是值传递，但是它的值(`Name`字段)并没有被修改，我们想改成`李四`，发现最后的结果还是`张三`。
>
> 这也就是说，`map`类型和我们自己定义的`struct`类型是不一样的。我们尝试把`modify`函数的接收参数改为`Person`的指针。
>
> ```go
> func main() {
> 	p:=Person{"张三"}
> 	modify(&p)
> 	fmt.Println(p)
> }
> 
> type Person struct {
> 	Name string
> }
> 
>  func modify(p *Person) {
> 	 p.Name = "李四"
>  }
> ```
>
> 在运行查看输出，我们发现，这次被修改了。我们这里省略了内存地址的打印，因为我们上面`int`类型的例子已经证明了指针类型的参数也是值传递的。 指针类型可以修改，非指针类型不行，那么我们可以大胆的猜测，我们使用`make`函数创建的`map`是不是一个指针类型呢？看一下源代码:
>
> ```go
> // makemap implements a Go map creation make(map[k]v, hint)
> // If the compiler has determined that the map or the first bucket
> // can be created on the stack, h and/or bucket may be non-nil.
> // If h != nil, the map can be created directly in h.
> // If bucket != nil, bucket can be used as the first bucket.
> func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap {
>     //省略无关代码
> }
> ```
>
> 通过查看`src/runtime/hashmap.go`源代码发现，的确和我们猜测的一样，`make`函数返回的是一个`hmap`类型的指针`*hmap`。也就是说`map===*hmap`。 现在看`func modify(p map)`这样的函数，其实就等于`func modify(p *hmap)`，和我们前面第一节**什么是值传递**里举的`func modify(ip *int)`的例子一样，可以参考分析。
>
> 所以在这里，Go语言通过`make`函数，字面量的包装，为我们省去了指针的操作，让我们可以更容易的使用map。这里的`map`可以理解为引用类型，但是记住引用类型不是传引用。

###### 4.4 chan类型

> `chan`类型本质上和`map`类型是一样的，这里不做过多的介绍，参考下源代码:
>
> ```go
> func makechan(t *chantype, size int64) *hchan {
>     //省略无关代码
> }
> ```
>
> `chan`也是一个引用类型，和`map`相差无几，`make`返回的是一个`*hchan`。

###### 4.5 slice

> `slice`和`map`、`chan`都不太一样的，一样的是，它也是引用类型，它也可以在函数中修改对应的内容。
>
> ```
> func main() {
> 	ages:=[]int{6,6,6}
> 	fmt.Printf("原始slice的内存地址是%p\n",ages)
> 	modify(ages)
> 	fmt.Println(ages)
> }
> 
> func modify(ages []int){
> 	fmt.Printf("函数里接收到slice的内存地址是%p\n",ages)
> 	ages[0]=1
> }
> ```
>
> 运行打印结果，发现的确是被修改了，而且我们这里打印`slice`的内存地址是可以直接通过`%p`打印的,不用使用`&`取地址符转换。
>
> 这就可以证明`make`的slice也是一个指针了吗？不一定，也可能`fmt.Printf`把`slice`特殊处理了。
>
> ```go
> func (p *pp) fmtPointer(value reflect.Value, verb rune) {
> 	var u uintptr
> 	switch value.Kind() {
> 	case reflect.Chan, reflect.Func, reflect.Map, reflect.Ptr, reflect.Slice, reflect.UnsafePointer:
> 		u = value.Pointer()
> 	default:
> 		p.badVerb(verb)
> 		return
> 	}
> 	//省略部分代码
> }
> ```
>
> 通过源代码发现，对于`chan`、`map`、`slice`等被当成指针处理，通过`value.Pointer()`获取对应的值的指针。
>
> ```go
> // If v's Kind is Slice, the returned pointer is to the first
> // element of the slice. If the slice is nil the returned value
> // is 0.  If the slice is empty but non-nil the return value is non-zero.
> func (v Value) Pointer() uintptr {
> 	// TODO: deprecate
> 	k := v.kind()
> 	switch k {
> 	//省略无关代码
> 	case Slice:
> 		return (*SliceHeader)(v.ptr).Data
> 	}
> }
> ```
>
> 很明显了，当是`slice`类型的时候，返回是`slice`这个结构体里，字段Data第一个元素的地址。
>
> ```go
> type SliceHeader struct {
> 	Data uintptr
> 	Len  int
> 	Cap  int
> }
> 
> type slice struct {
> 	array unsafe.Pointer
> 	len   int
> 	cap   int
> }
> ```
>
> 所以我们通过`%p`打印的`slice`变量`ages`的地址其实就是内部存储数组元素的地址，`slice`是一种结构体+元素指针的混合类型，通过元素`array`(`Data`)的指针，可以达到修改`slice`里存储元素的目的。
>
> 所以修改类型的内容的办法有很多种，类型本身作为指针可以，类型里有指针类型的字段也可以。
>
> 单纯的从`slice`这个结构体看，我们可以通过`modify`修改存储元素的内容，但是永远修改不了`len`和`cap`，因为他们只是一个拷贝，如果要修改，那就要传递`*slice`作为参数才可以。
>
> ```go
> func main() {
> 	i:=19
> 	p:=Person{name:"张三",age:&i}
> 	fmt.Println(p)
> 	modify(p)
> 	fmt.Println(p)
> }
> 
> type Person struct {
> 	name string
> 	age  *int
> }
> 
> func (p Person) String() string{
> 	return "姓名为：" + p.name + ",年龄为："+ strconv.Itoa(*p.age)
> }
> 
> func modify(p Person){
> 	p.name = "李四"
> 	*p.age = 20
> }
> ```
>
> 运行打印输出结果为：
>
> ```
> 姓名为：张三,年龄为：19
> 姓名为：张三,年龄为：20
> ```
>
> 通过这个`Person`和`slice`对比，就更好理解了，`Person`的`name`字段就类似于`slice`的`len`和`cap`字段，`age`字段类似于`array`字段。在传参为非指针类型的情况下，只能修改`age`字段，`name`字段无法修改。要修改`name`字段，就要把传参改为指针，比如：
>
> ```go
> modify(&p)
> func modify(p *Person){
> 	p.name = "李四"
> 	*p.age = 20
> }
> ```
>
> 这样`name`和`age`字段双双都被修改了。
>
> 所以`slice`类型也是引用类型。

###### 4.6 小结

> 最终我们可以确认的是Go语言中所有的传参都是值传递（传值），都是一个副本，一个拷贝。因为拷贝的内容有时候是非引用类型（int、string、struct等这些），这样就在函数中就无法修改原内容数据；有的是引用类型（指针、map、slice、chan等这些），这样就可以修改原内容数据。
>
> 是否可以修改原内容数据，和传值、传引用没有必然的关系。在C++中，传引用肯定是可以修改原内容数据的，在Go语言里，虽然只有传值，但是我们也可以修改原内容数据，因为参数是引用类型。
>
> 这里也要记住，引用类型和传引用是两个概念。
>
> 再记住，Go里只有传值（值传递）。

##### 5. 切片

###### 5.1 切片的三种状态

> 一个非常细节的小知识，这个知识被大多数 Go 语言的开发者无视了，它就是切片的三种特殊状态 —— 「零切片」、「空切片」和「nil 切片」。
>
> ![深度解析 Go 语言「切片」的三种特殊状态](http://p9.pstatp.com/large/pgc-image/e02320dba4fa4a82b33d9988c29c0eaf)
>
> 切片被视为 Go 语言中最为重要的基础数据结构，使用起来非常简单，有趣的内部结构让它成了 Go 语言面试中最为常见的考点。切片的底层是一个数组，切片的表层是一个包含三个变量的结构体，当我们将一个切片赋值给另一个切片时，本质上是对切片表层结构体的**浅拷贝**。结构体中第一个变量是一个指针，指向底层的数组，另外两个变量分别是切片的长度和容量。
>
> ![深度解析 Go 语言「切片」的三种特殊状态](http://p3.pstatp.com/large/pgc-image/e9dacc1f05d343d4a1f93264c6eab77e)
>
> 我们今天要讲的特殊状态之一「零切片」其实并不是什么特殊的切片，它只是表示底层数组的二进制内容都是零。比如下面代码中的 s 变量就是一个「零切片」
>
> ![深度解析 Go 语言「切片」的三种特殊状态](http://p3.pstatp.com/large/pgc-image/938fda8ad4e2437eb8604d366462fe6b)
>
> 如果是一个指针类型的切片，那么底层数组的内容就全是 nil
>
> 
>
> ![深度解析 Go 语言「切片」的三种特殊状态](http://p9.pstatp.com/large/pgc-image/7e5a0db398074d5ca7c6675e953c5902)零切片还是比较易于理解的，这部分我也就不再以钻牛角尖的形式继续自我拷问。 下面我们要引入「空切片」和 「nil 切片」，在理解它们的区别之前我们先看看一个长度为零的切片都有那些形式可以创建出来
>
> ![深度解析 Go 语言「切片」的三种特殊状态](http://p1.pstatp.com/large/pgc-image/64333df6fd4b482599937fb8584734a4)
>
> 上面这四种形式从输出结果上来看，似乎一摸一样，没区别。但是实际上是有区别的，我们要讲的两种特殊类型「空切片」和「 nil 切片」，就隐藏在上面的四种形式之中。 我们如何来分析三面四种形式的内部结构的区别呢？接下里要使用到 Go 语言的高级内容，通过 unsafe.Pointer 来转换 Go 语言的任意变量类型。 因为切片的内部结构是一个结构体，包含三个机器字大小的整型变量，其中第一个变量是一个指针变量，指针变量里面存储的也是一个整型值，只不过这个值是另一个变量的内存地址。我们可以将这个结构体看成长度为 3 的整型数组 [3]int。然后将切片变量转换成 [3]int。
>
> ![深度解析 Go 语言「切片」的三种特殊状态](http://p1.pstatp.com/large/pgc-image/b628194daae94aaf8389cb2bc3d34c03)
>
> 从输出中我们看到了明显的神奇的让人感到意外的难以理解的不一样的结果。 其中输出为 [0 0 0] 的 s1 和 s4 变量就是「 nil 切片」，s2 和 s3 变量就是「空切片」。824634199592 这个值是一个特殊的内存地址，所有类型的「空切片」都共享这一个内存地址。
>
> ![深度解析 Go 语言「切片」的三种特殊状态](http://p99.pstatp.com/large/pgc-image/2e14e64ab55a4e688b039ccc2d85d938)
>
> 用图形来表示「空切片」和「 nil 切片」如下
>
> ![深度解析 Go 语言「切片」的三种特殊状态](http://p9.pstatp.com/large/pgc-image/5c2ec147981d4ffca3f36c50dd5ecfab)
>
> 空切片指向的 zerobase 内存地址是一个神奇的地址，从 Go 语言的源代码中可以看到它的定义
>
> ![深度解析 Go 语言「切片」的三种特殊状态](http://p3.pstatp.com/large/pgc-image/818f52beb2534218badfe86781cf09c1)
>
> 最后一个问题是：「 nil 切片」和 「空切片」在使用上有什么区别么？ 答案是完全没有任何区别！No！不对，还有一个小小的区别！请看下面的代码
>
> ![深度解析 Go 语言「切片」的三种特殊状态](http://p99.pstatp.com/large/pgc-image/3cead424a5fe4ea780a3fbf6d0dd6591)
>
> 所以为了避免写代码的时候把脑袋搞昏的最好办法是不要创建「 空切片」，统一使用「 nil 切片」，同时要避免将切片和 nil 进行比较来执行某些逻辑。这是官方的标准建议。
>
> 「空切片」和「 nil 切片」有时候会隐藏在结构体中，这时候它们的区别就被太多的人忽略了，下面我们看个例子
>
> ![深度解析 Go 语言「切片」的三种特殊状态](http://p9.pstatp.com/large/pgc-image/d9636524219a4fdc8d15deb5341cdb28)
>
> 可以发现这两种创建结构体的结果是不一样的！ 「空切片」和「 nil 切片」还有一个极为不同的地方在于 JSON 序列化
>
> ![深度解析 Go 语言「切片」的三种特殊状态](http://p3.pstatp.com/large/pgc-image/1d6b583d3f1e47a6ba25db8f68d07d15)





#### 二. 包的总结          

##### 1. JSON

>  json格式可以算我们日常最常用的序列化格式之一了，Go语言作为一个由Google开发，号称互联网的C语言的语言，自然也对JSON格式支持很好。但是Go语言是个强类型语言，对格式要求极其严格而JSON格式虽然也有类型，但是并不稳定，Go语言在解析来源为非强类型语言时比如PHP等序列化的JSON时，经常遇到一些问题诸如字段类型变化导致无法正常解析的情况，导致服务不稳定。所以本篇的主要目的
>
> 1. 就是挖掘Golang解析json的绝大部分能力
> 2. 比较优雅的解决解析json时存在的各种问题
> 3. 深入一下Golang解析json的过程

- ###### Golang解析JSON之Tag篇

1. 一个结构体正常序列化过后是什么样的呢？

```go
package main
import (
    "encoding/json"
    "fmt"
)
 
// Product 商品信息
type Product struct {
    Name      string
    ProductID int64
    Number    int
    Price     float64
    IsOnSale  bool
}
 
func main() {
    p := &Product{}
    p.Name = "Xiao mi 6"
    p.IsOnSale = true
    p.Number = 10000
    p.Price = 2499.00
    p.ProductID = 1
    data, _ := json.Marshal(p)
    fmt.Println(string(data))
}
 
 
//结果
{"Name":"Xiao mi 6","ProductID":1,"Number":10000,"Price":2499,"IsOnSale":true}
```

2. 何为Tag，tag就是标签，给结构体的每个字段打上一个标签，标签冒号前是类型，后面是标签名

```go
// Product _
type Product struct {
    Name      string  `json:"name"`
    ProductID int64   `json:"-"` // 表示不进行序列化
    Number    int     `json:"number"`
    Price     float64 `json:"price"`
    IsOnSale  bool    `json:"is_on_sale,string"`
}
 
// 序列化过后，可以看见
   {"name":"Xiao mi 6","number":10000,"price":2499,"is_on_sale":"false"}
```

3. omitempty，tag里面加上omitempy，可以在序列化的时候忽略0值或者空值

```go
package main
 
import (
    "encoding/json"
    "fmt"
)
 
// Product _
type Product struct {
    Name      string  `json:"name"`
    ProductID int64   `json:"product_id,omitempty"`
    Number    int     `json:"number"`
    Price     float64 `json:"price"`
    IsOnSale  bool    `json:"is_on_sale,omitempty"`
}
 
func main() {
    p := &Product{}
    p.Name = "Xiao mi 6"
    p.IsOnSale = false
    p.Number = 10000
    p.Price = 2499.00
    p.ProductID = 0
 
    data, _ := json.Marshal(p)
    fmt.Println(string(data))
}
// 结果
{"name":"Xiao mi 6","number":10000,"price":2499}
```

　　4. type，有些时候，我们在序列化或者反序列化的时候，可能结构体类型和需要的类型不一致，这个时候可以指定,支持string,number和boolean

```go
package main
 
import (
    "encoding/json"
    "fmt"
)
 
// Product _
type Product struct {
    Name      string  `json:"name"`
    ProductID int64   `json:"product_id,string"`
    Number    int     `json:"number,string"`
    Price     float64 `json:"price,string"`
    IsOnSale  bool    `json:"is_on_sale,string"`
}
 
func main() {
 
    var data = `{"name":"Xiao mi 6","product_id":"10","number":"10000","price":"2499","is_on_sale":"true"}`
    p := &Product{}
    err := json.Unmarshal([]byte(data), p)
    fmt.Println(err)
    fmt.Println(*p)
}
// 结果
<nil>
{Xiao mi 6 10 10000 2499 true}
```

- **下面讲一讲Golang如何自定义解析JSON，Golang自带的JSON解析功能非常强悍**

###### 说明

> 很多时候，我们可能遇到这样的场景，就是远端返回的JSON数据不是你想要的类型，或者你想做额外的操作，比如在解析的过程中进行校验，或者类型转换，那么我们可以这样或者在解析过程中进行数据转换

###### 实例

```go
package main
 
import (
    "bytes"
    "encoding/json"
    "fmt"
)
 
// Mail _
type Mail struct {
    Value string
}
 
// UnmarshalJSON _
func (m *Mail) UnmarshalJSON(data []byte) error {
    // 这里简单演示一下，简单判断即可
    if bytes.Contains(data, []byte("@")) {
        return fmt.Errorf("mail format error")
    }
    m.Value = string(data)
    return nil
}
 
// UnmarshalJSON _
func (m *Mail) MarshalJSON() (data []byte, err error) {
    if m != nil {
        data = []byte(m.Value)
    }
    return
}
 
// Phone _
type Phone struct {
    Value string
}
 
// UnmarshalJSON _
func (p *Phone) UnmarshalJSON(data []byte) error {
    // 这里简单演示一下，简单判断即可
    if len(data) != 11 {
        return fmt.Errorf("phone format error")
    }
    p.Value = string(data)
    return nil
}
 
// UnmarshalJSON _
func (p *Phone) MarshalJSON() (data []byte, err error) {
    if p != nil {
        data = []byte(p.Value)
    }
    return
}
 
// UserRequest _
type UserRequest struct {
    Name  string
    Mail  Mail
    Phone Phone
}
 
func main() {
    user := UserRequest{}
    user.Name = "ysy"
    user.Mail.Value = "yangshiyu@x.com"
    user.Phone.Value = "18900001111"
    fmt.Println(json.Marshal(user))
}
```

　　

###### 为什么要这样？

如果是客户端开发，需要开发大量的API，接收大量的JSON，在开发早期定义各种类型看起来是很大的工作量，不如写 if else  判断数据简单暴力。但是到开发末期，你会发现预先定义的方式能极大的提高你的代码质量，减少代码量。下面实例1和实例2，谁能减少代码一目了然

```go
实例1，if else做数据校验
// UserRequest _
type UserRequest struct {
    Name  string
    Mail  string
    Phone string
}
func AddUser(data []byte) (err error) {
    user := &UserRequest{}
    err = json.Unmarshal(data, user)
    if err != nil {
        return
    }
    //
    if isMail(user.Mail) {
        return fmt.Errorf("mail format error")
    }
 
    if isPhone(user.Phone) {
        return fmt.Errorf("phone format error")
    }
 
    // TODO
    return
}
 
实例2，利用预先定义好的类型，在解析时就进行判断
// UserRequest _
type UserRequest struct {
    Name  string
    Mail  Mail
    Phone Phone
}
 
func AddUser(data []byte) {
    user := &UserRequest{}
    err = json.Unmarshal(data, user)
    if err != nil {
        return
    }
 
    // TODO
 
}
```

* ##### 第三方库

  >  Json 作为一种重要的数据格式，具有良好的可读性以及自描述性，广泛地应用在各种数据传输场景中。Go  语言里面原生支持了这种数据格式的序列化以及反序列化，内部使用反射机制实现，性能有点差，在高度依赖 json  解析的应用里，往往会成为性能瓶颈，好在已有很多第三方库帮我们解决了这个问题，但是这么多库，对于像我这种有选择困难症的人来说，到底要怎么选择呢，下面就给大家来一一分析一下

  ### ffjson

  ```go
  go get -u github.com/pquerna/ffjson
  ```

  > 原生的库性能比较差的主要原因是使用了很多反射的机制，为了解决这个问题，ffjson 通过预编译生成代码，类型的判断在预编译阶段已经确定，避免了在运行时的反射

  > 但也因此在编译前需要多一个步骤，需要先生成 ffjson 代码，生成代码只需要执行 `ffjson ` 就可以了，其中 `file.go` 是一个包含 json 结构体定义的 go 文件。注意这里 ffjson 是这个库提供的一个代码生成工具，直接执行上面的 `go get` 会把这个工具安装在 `$GOPATH/bin` 目录下，把 `$GOPATH/bin` 加到 `$PATH` 环境变量里面，可以全局访问

  另外，如果有些结构，不想让 ffjson 生成代码，可以通过增加注释的方式

  ```go
  // ffjson: skip
  type Foo struct {
     Bar string
  }
  
  // ffjson: nodecoder
  type Foo struct {
     Bar string
  }
  ```

  ### easyjson

  ```
  go get -u github.com/mailru/easyjson/...
  ```

  > easyjson 的思想和 ffjson 是一致的，都是增加一个预编译的过程，预先生成对应结构的序列化反序列化代码，除此之外，easyjson 还放弃了一些原生库里面支持的一些不必要的特性，比如：key 类型声明，key 大小写不敏感等等，以达到更高的性能

  生成代码执行 `easyjson -all ` 即可，如果不指定 `-all` 参数，只会对带有 `//easyjson:json` 的结构生成代码

  ```go
  //easyjson:json
  type A struct {
      Bar string
  }
  ```

  ### jsoniter

  ```go
  go get -u github.com/json-iterator/go
  ```

  > 这是一个很神奇的库，滴滴开发的，不像 easyjson 和 ffjson 都使用了预编译，而且 100% 兼容原生库，但是性能超级好，也不知道怎么实现的，如果有人知道的话，可以告诉我一下吗？

  使用上面，你只要把所有的

  ```go
  import "encoding/json"
  ```

  替换成

  ```go 
  import "github.com/json-iterator/go"
  
  var json = jsoniter.ConfigCompatibleWithStandardLibrary
  ```

  就可以了，其它都不需要动

  ### codec-json

  ```go
  go get -u github.com/ugorji/go/codec
  ```

  这个库里面其实包含很多内容，json 只是其中的一个功能，比较老，使用起来比较麻烦，性能也不是很好

  ### jsonparser

  ```go
  go get -u github.com/buger/jsonparser
  ```

  >  严格来说，这个库不属于 json 序列化的库，只是提供了一些 json 解析的接口，使用的时候需要自己去设置结构里面的值，事实上，每次调用都需要重新解析 json 对象，性能并不是很好

  就像名字暗示的那样，这个库只是一个解析库，并没有序列化的接口

  ### 性能测试

  对上面这些 json 库，作了一些性能测试，测试代码在：[https://github.com/hatlonely/...](https://github.com/hatlonely/hellogolang/blob/master/internal/json/json_benchmark_test.go)，下面是在我的 Macbook 上测试的结果（实际结果和库的版本以及机器环境有关，建议自己再测试一遍）：

  ```
  BenchmarkMarshalStdJson-4                    1000000          1097 ns/op
  BenchmarkMarshalJsonIterator-4               2000000           781 ns/op
  BenchmarkMarshalFfjson-4                     2000000           941 ns/op
  BenchmarkMarshalEasyjson-4                   3000000           513 ns/op
  BenchmarkMarshalCodecJson-4                  1000000          1074 ns/op
  BenchmarkMarshalCodecJsonWithBufio-4         1000000          2161 ns/op
  BenchmarkUnMarshalStdJson-4                   500000          2512 ns/op
  BenchmarkUnMarshalJsonIterator-4             2000000           591 ns/op
  BenchmarkUnMarshalFfjson-4                   1000000          1127 ns/op
  BenchmarkUnMarshalEasyjson-4                 2000000           608 ns/op
  BenchmarkUnMarshalCodecJson-4                  20000        122694 ns/op
  BenchmarkUnMarshalCodecJsonWithBufio-4        500000          3417 ns/op
  BenchmarkUnMarshalJsonparser-4               2000000           877 ns/op
  ```

  从上面的结果可以看出来：

  1. easyjson 无论是序列化还是反序列化都是最优的，序列化提升了1倍，反序列化提升了3倍
  2. jsoniter 性能也很好，接近于easyjson，关键是没有预编译过程，100%兼容原生库
  3. ffjson 的序列化提升并不明显，反序列化提升了1倍
  4. codecjson 和原生库相比，差不太多，甚至更差
  5. jsonparser 不太适合这样的场景，性能提升并不明显，而且没有反序列化

  **所以综合考虑，建议大家使用 jsoniter，如果追求极致的性能，考虑 easyjson**

  ### 参考链接

  ffjson: [https://github.com/pquerna/ff...](https://github.com/pquerna/ffjson)
  easyjson: [https://github.com/mailru/eas...](https://github.com/mailru/easyjson)
  jsoniter: [https://github.com/json-itera...](https://github.com/json-iterator/go)
  jsonparser: [https://github.com/buger/json...](https://github.com/buger/jsonparser)
  codecjson: [http://ugorji.net/blog/go-cod...](http://ugorji.net/blog/go-codec-primer)





##### 2. 文件/文件夹的相关操作

###### 2.1 检查指定文件是否存在

```go
func FileIsExisted(filename string) bool {
	existed := true
	if _, err := os.Stat(filename); os.IsNotExist(err) {
		existed = false
	}
	return existed
}
```

###### 2.2 检查指定文件夹是否存在

```go
func IsDir(name string) bool {
	if info, err := os.Stat(name); err == nil {
		return info.IsDir()
	}
	return false
}
```

###### 2.3 创建文件夹

```go
func MakeDir(dir string) error {
	if !FileIsExisted(dir) {
		if err := os.MkdirAll(dir, 0777); err != nil { //os.ModePerm
			fmt.Println("MakeDir failed:", err)
			return err
		}
	}
	return nil
}
```

###### 2.4 复制文件

```go
//使用io.Copy
func CopyFile(src, des string) (written int64, err error) {
	srcFile, err := os.Open(src)
	if err != nil {
		return 0, err
	}
	defer srcFile.Close()
 
	//获取源文件的权限
	fi, _ := srcFile.Stat()
	perm := fi.Mode()
 
	//desFile, err := os.Create(des)  //无法复制源文件的所有权限
	desFile, err := os.OpenFile(des, os.O_RDWR|os.O_CREATE|os.O_TRUNC, perm)  //复制源文件的所有权限
	if err != nil {
		return 0, err
	}
	defer desFile.Close()
 
	return io.Copy(desFile, srcFile)
}
 
 
//使用ioutil.WriteFile()和ioutil.ReadFile()
func CopyFile2(src, des string) (written int64, err error) {
	//获取源文件的权限
	srcFile, err := os.Open(src)
	if err != nil {
		return 0, err
	}
	fi, _ := srcFile.Stat()
	perm := fi.Mode()
	srcFile.Close()
 
	input, err := ioutil.ReadFile(src)
	if err != nil {
		return 0, err
	}
 
	err = ioutil.WriteFile(des, input, perm)
	if err != nil {
		return 0, err
	}
 
	return int64(len(input)), nil
}
 
 
//使用os.Read()和os.Write()
func CopyFile3(src, des string, bufSize int) (written int64, err error) {
	if bufSize <= 0 {
		bufSize = 1*1024*1024   //1M
	}
	buf := make([]byte, bufSize)
 
	srcFile, err := os.Open(src)
	if err != nil {
		return 0, err
	}
	defer srcFile.Close()
 
	//获取源文件的权限
	fi, _ := srcFile.Stat()
	perm := fi.Mode()
 
	desFile, err := os.OpenFile(des, os.O_CREATE|os.O_RDWR|os.O_TRUNC, perm)
	if err != nil {
		return 0, err
	}
	defer desFile.Close()
 
	count := 0
	for {
		n, err := srcFile.Read(buf)
		if err != nil && err != io.EOF {
			return 0, err
		}
 
		if n == 0 {
			break
		}
 
		if wn, err := desFile.Write(buf[:n]); err != nil {
			return 0, err
		} else {
			count += wn
		}
	}
 
	return int64(count), nil
}
```

###### 2.5 复制整个文件夹

```go
func CopyDir(srcPath, desPath string) error {
	//检查目录是否正确
	if srcInfo, err := os.Stat(srcPath); err != nil {
		return err
	} else {
		if !srcInfo.IsDir() {
			return errors.New("源路径不是一个正确的目录！")
		}
	}
 
	if desInfo, err := os.Stat(desPath); err != nil {
		return err
	} else {
		if !desInfo.IsDir() {
			return errors.New("目标路径不是一个正确的目录！")
		}
	}
 
	if strings.TrimSpace(srcPath) == strings.TrimSpace(desPath) {
		return errors.New("源路径与目标路径不能相同！")
	}
 
	err := filepath.Walk(srcPath, func(path string, f os.FileInfo, err error) error {
		if f == nil {
			return err
		}
 
		//复制目录是将源目录中的子目录复制到目标路径中，不包含源目录本身
		if path == srcPath {
			return nil
		}
 
		//生成新路径
		destNewPath := strings.Replace(path, srcPath, desPath, -1)
 
		if !f.IsDir() {
			CopyFile(path, destNewPath)
		} else {
			if !FileIsExisted(destNewPath) {
				return MakeDir(destNewPath)
			}
		}
 
		return nil
	})
 
	return err
}
```

###### 2.6 遍历指定文件夹中的所有文件（不进入下一级子目录）

```go
/* 获取指定路径下的所有文件，只搜索当前路径，不进入下一级目录，可匹配后缀过滤（suffix为空则不过滤）*/
func ListDir(dir, suffix string) (files []string, err error) {
   files = []string{}
 
   _dir, err := ioutil.ReadDir(dir)
   if err != nil {
      return nil, err
   }
 
   suffix = strings.ToLower(suffix)  //匹配后缀
 
   for _, _file := range _dir {
      if _file.IsDir() {
         continue   //忽略目录
      }
      if len(suffix) == 0 || strings.HasSuffix(strings.ToLower(_file.Name()), suffix) {
         //文件后缀匹配
         files = append(files, path.Join(dir, _file.Name()))
      }
   }
 
   return files, nil
}
```

###### 2.7 遍历指定路径文件夹及子目录中的所有文件

```go
 
/* 获取指定路径下以及所有子目录下的所有文件，可匹配后缀过滤（suffix为空则不过滤）*/
func WalkDir(dir, suffix string) (files []string, err error) {
	files = []string{}
 
	err = filepath.Walk(dir, func(fname string, fi os.FileInfo, err error) error {
		if fi.IsDir() {
			//忽略目录
			return nil
		}
 
		if len(suffix) == 0 || strings.HasSuffix(strings.ToLower(fi.Name()), suffix) {
			//文件后缀匹配
			files = append(files, fname)
		}
 
		return nil
	})
 
	return files, err
}
```

######  2.8 删除文件

```go
os.Remove(filename)
```

###### 2.9 删除文件夹

```go
os.RemoveAll(dir)
```

###### 2.10 复制嵌套文件夹

```go
//待测试
func FormatPath(s string) string {
	switch runtime.GOOS {
	case "windows":
		return strings.Replace(s, "/", "\\", -1)
	case "darwin", "linux":
		return strings.Replace(s, "\\", "/", -1)
	default:
		logger.Println("only support linux,windows,darwin, but os is " + runtime.GOOS)
		return s
	}
}

func copyDir(src string, dest string) {
	src = FormatPath(src)
	dest = FormatPath(dest)
	log.Println(src)
	log.Println(dest)

	var cmd *exec.Cmd

	switch runtime.GOOS {
	case "windows":
		cmd = exec.Command("xcopy", src, dest, "/I", "/E")
	case "darwin", "linux":
		cmd = exec.Command("cp", "-R", src, dest)
	}

	outPut, e := cmd.Output()
	if e != nil {
		logger.Println(e.Error())
		return
	}
	fmt.Println(string(outPut))
}

func main() {
    // windows
    copyDir("d:/tmp","d:/tmp2")
    // linux, mac
    copyDir("/home/tmp", "/home/tmp2")
}

```



##### 3. zip 压缩文件的相关操作

###### 3.1 需要导入的包

```go

 import (
	"archive/zip"
	"bytes"
	"io"
	"os"
	"path/filepath"
)
```



###### 3.2 判断是不是压缩文件

```go
func isZip(zipPath string) bool {
	f, err := os.Open(zipPath)
	if err != nil {
		return false
	}
	defer f.Close()
 
	buf := make([]byte, 4)
	if n, err := f.Read(buf); err != nil || n < 4 {
		return false
	}
 
	return bytes.Equal(buf, []byte("PK\x03\x04"))
}
```

###### 3.3 解压文件

```go
func unzip(archive, target string) error {
	reader, err := zip.OpenReader(archive)
	if err != nil {
		return err
	}
 
	if err := os.MkdirAll(target, 0755); err != nil {
		return err
	}
 
	for _, file := range reader.File {
		path := filepath.Join(target, file.Name)
		if file.FileInfo().IsDir() {
			os.MkdirAll(path, file.Mode())
			continue
		}
		//------------注入
 
		dir := filepath.Dir(path)
		if len(dir) > 0 {
			if _, err = os.Stat(dir); os.IsNotExist(err) {
				err = os.MkdirAll(dir, 0755)
				if err != nil {
					return err
				}
			}
		}
 
		//---------------------end
 
		fileReader, err := file.Open()
		if err != nil {
			return err
		}
		defer fileReader.Close()
 
		targetFile, err := os.OpenFile(path, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, file.Mode())
		if err != nil {
			return err
		}
		defer targetFile.Close()
 
		if _, err := io.Copy(targetFile, fileReader); err != nil {
			return err
		}
	}
 
	return nil
}
```

###### 3.4 压缩文件

```go
//压缩文件
//files 文件数组，可以是不同dir下的文件或者文件夹
//dest 压缩文件存放地址
func Compress(files []*os.File, dest string) error {
    //func Create(name string) (file *File, err error)
    /*
    Create采用模式0666（任何人都可读写，不可执行）创建一个名为name的文件，如果文件已存在会截断它（为空文件）。如果成功，返回的文件对象可用于I/O；对应的文件描述符具有O_RDWR模式。如果出错，错误底层类型是*PathError。
    */
	d, _ := os.Create(dest)
	defer d.Close()
    //func NewWriter(w io.Writer) *Writer
    /*
    	NewWriter创建并返回一个将zip文件写入w的*Writer。
    */
	w := zip.NewWriter(d)
	defer w.Close()
	for _, file := range files {
		err := compress(file, "", w)
		if err != nil {
			return err
		}
	}
	return nil
}
 
func compress(file *os.File, prefix string, zw *zip.Writer) error {
    //Stat返回描述文件f的FileInfo类型值。如果出错，错误底层类型是*PathError。
    /*
    type FileInfo interface {
        Name() string       // 文件的名字（不含扩展名）
        Size() int64        // 普通文件返回值表示其大小；其他文件的返回值含义各系统不同
        Mode() FileMode     // 文件的模式位
        ModTime() time.Time // 文件的修改时间
        IsDir() bool        // 等价于Mode().IsDir()，判断该文件是不是一个文件夹
        Sys() interface{}   // 底层数据来源（可以返回nil）
    }
    */
	info, err := file.Stat()
	if err != nil {
		return err
	}
	if info.IsDir() {
		if len(prefix) == 0 {
			prefix = info.Name()
		} else {
			prefix = prefix + "/" + info.Name()
		}
        /*
        	func (f *File) Readdir(n int) (fi []FileInfo, err error)
        	Readdir读取目录f的内容，返回一个有n个成员的[]FileInfo，这些FileInfo是被Lstat返回的，采用		目录顺序。对本函数的下一次调用会返回上一次调用剩余未读取的内容的信息。

			如果n>0，Readdir函数会返回一个最多n个成员的切片。这时，如果Readdir返回一个空切片，它会返回		  一个非nil的错误说明原因。如果到达了目录f的结尾，返回值err会是io.EOF。

			如果n<=0，Readdir函数返回目录中剩余所有文件对象的FileInfo构成的切片。此时，如果Readdir调用		  成功（读取所有内容直到结尾），它会返回该切片和nil的错误值。如果在到达结尾前遇到错误，会返回之前成功		 读取的FileInfo构成的切片和该错误。
        
        */
		fileInfos, err := file.Readdir(-1)
		if err != nil {
			return err
		}
		for _, fi := range fileInfos {
            //func OpenFile(name string, flag int, perm FileMode) (file *File, err error)
           /*
           OpenFile是一个更一般性的文件打开函数，大多数调用者都应用Open或Create代替本函数。它会使用指定		的选项（如O_RDONLY等）、指定的模式（如0666等）打开指定名称的文件。如果操作成功，返回的文件对象可用于	  I/O。如果出错，错误底层类型是*PathError
           */
            //func Open(name string) (file *File, err error)
            /*
            Open打开一个文件用于读取。如果操作成功，返回的文件对象的方法可用于读取数据；对应的文件描述符具		有O_RDONLY模式。如果出错，错误底层类型是*PathError。
            */
			f, err := os.Open(file.Name() + "/" + fi.Name())
			if err != nil {
				return err
			}
			err = compress(f, prefix, zw)
			if err != nil {
				return err
			}
		}
	} else {
        //func FileInfoHeader(fi os.FileInfo) (*FileHeader, error)
        /*
        FileInfoHeader返回一个根据fi填写了部分字段的Header。因为os.FileInfo接口的Name方法只返回它描	 述的文件的无路径名，有可能需要将返回值的Name字段修改为文件的完整路径名。
        */
		header, err := zip.FileInfoHeader(info)
		if len(prefix) == 0 {
			header.Name = header.Name
		} else {
			header.Name = prefix + "/" + header.Name
		}
		if err != nil {
			return err
		}
        
        //func (w *Writer) CreateHeader(fh *FileHeader) (io.Writer, error)
        /*
        使用给出的*FileHeader来作为文件的元数据添加一个文件进zip文件。本方法返回一个io.Writer接口（用于		写入新添加文件的内容）。新增文件的内容必须在下一次调用CreateHeader、Create或Close方法之前全部写入。
        */
		writer, err := zw.CreateHeader(header)
		if err != nil {
			return err
		}
		_, err = io.Copy(writer, file)
		file.Close()
		if err != nil {
			return err
		}
	}
	return nil
}
```

###### 3.5 调用方式

```go
  //解压：  
  unzip("/img_data/2018-07-21.zip","/img_data/")
  
  
 //压缩 
 f, err := os.Open("/img_data/2018-07-21")
 if err != nil {
     fmt.Println(err)
 }
  defer f.Close()
  var files = []*os.File{f}
  err = Compress(files, "/img_data/2018-07-21.zip")
  if err != nil {
      fmt.Println(err)
  }
```

##### 4. net/http的使用

###### 4.1 搭建http服务器

```go
func main() {
	//开启服务器
	//共有两种方法
	/*
		1. http.Handle("/foo", fooHandler)
		2.
	*/

	//第一种方式开启一个监听服务器
	http.Handle("/test1",http.HandlerFunc(myhandler))


	//第二中方式
	http.HandleFunc("/test2", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
	})

	http.ListenAndServe(":8080",nil)

}

func myhandler(rw http.ResponseWriter, req *http.Request) {
	req.ParseForm()
	fmt.Println(req.Form)
	fmt.Println("path", req.URL.Path)
	fmt.Println("scheme", req.URL.Scheme)
	fmt.Println(req.Form["url_long"])
	for k, v := range req.Form {
		fmt.Println("key:", k)
		fmt.Println("val:", strings.Join(v, ""))
	}
	fmt.Fprintf(rw, "Hello	abs!")
}
```

###### 4.2 搭建http客户端

* **直接发送请求**

```go
//发送请求


//func Get(url string) (resp *Response, err error)
/*
Get向指定的URL发出一个GET请求，如果回应的状态码如下，Get会在调用c.CheckRedirect后执行重定向：
    301 (Moved Permanently)
    302 (Found)
    303 (See Other)
    307 (Temporary Redirect)
如果c.CheckRedirect执行失败或存在HTTP协议错误时，本方法将返回该错误；如果回应的状态码不是2xx，本方法并不会返回错误。如果返回值err为nil，resp.Body总是非nil的，调用者应该在读取完resp.Body后关闭它。
*/
resp, err := http.Get("http://example.com/")

//func Post(url string, bodyType string, body io.Reader) (resp *Response, err error)
/*
Post向指定的URL发出一个POST请求。bodyType为POST数据的类型， body为POST数据，作为请求的主体。如果参数body实现了io.Closer接口，它会在发送请求后被关闭。调用者有责任在读取完返回值resp的主体后关闭它。

Post是对包变量DefaultClient的Post方法的包装。
*/
resp, err := http.Post("http://example.com/upload", "image/jpeg", &buf)

//func PostForm(url string, data url.Values) (resp *Response, err error)
/*
PostForm向指定的URL发出一个POST请求，url.Values类型的data会被编码为请求的主体。如果返回值err为nil，resp.Body总是非nil的，调用者应该在读取完resp.Body后关闭它。
*/
resp, err := http.PostForm("http://example.com/form",
	url.Values{"key": {"Value"}, "id": {"123"}})


//处理返回值
if err != nil {
	// handle error
}


//程序在使用完回复后必须关闭回复的主体。
defer resp.Body.Close()


body, err := ioutil.ReadAll(resp.Body)
if err != nil {
		return nil,err
	}


//检测返回值
result := make(map[string]interface{})
err = json.Unmarshal(body, &result)
if err != nil {
    return nil,err
}



```

* **构建Client**

```go
client := &http.Client{
	CheckRedirect: redirectPolicyFunc,
}
//resp, err := client.Get("http://example.com")
// ...
//func NewRequest(method, urlStr string, body io.Reader) (*Request, error)
/*
NewRequest使用指定的方法、网址和可选的主题创建并返回一个新的*Request。

如果body参数实现了io.Closer接口，Request返回值的Body 字段会被设置为body，并会被Client类型的Do、Post和PostFOrm方法以及Transport.RoundTrip方法关闭。
*/
req, err := http.NewRequest("GET", "http://example.com", nil)
// ...
req.Header.Add("If-None-Match", `W/"wyzzy"`)

resp, err := client.Do(req)
// ...
```

###### 4.3 Request

* **结构体**

```go
type Request struct {
    // Method指定HTTP方法（GET、POST、PUT等）。对客户端，""代表GET。
    Method string
    // URL在服务端表示被请求的URI，在客户端表示要访问的URL。
    //
    // 在服务端，URL字段是解析请求行的URI（保存在RequestURI字段）得到的，
    // 对大多数请求来说，除了Path和RawQuery之外的字段都是空字符串。
    // （参见RFC 2616, Section 5.1.2）
    //
    // 在客户端，URL的Host字段指定了要连接的服务器，
    // 而Request的Host字段（可选地）指定要发送的HTTP请求的Host头的值。
    URL *url.URL
    // 接收到的请求的协议版本。本包生产的Request总是使用HTTP/1.1
    Proto      string // "HTTP/1.0"
    ProtoMajor int    // 1
    ProtoMinor int    // 0
    // Header字段用来表示HTTP请求的头域。如果头域（多行键值对格式）为：
    //	accept-encoding: gzip, deflate
    //	Accept-Language: en-us
    //	Connection: keep-alive
    // 则：
    //	Header = map[string][]string{
    //		"Accept-Encoding": {"gzip, deflate"},
    //		"Accept-Language": {"en-us"},
    //		"Connection": {"keep-alive"},
    //	}
    // HTTP规定头域的键名（头名）是大小写敏感的，请求的解析器通过规范化头域的键名来实现这点。
    // 在客户端的请求，可能会被自动添加或重写Header中的特定的头，参见Request.Write方法。
    Header Header
    // Body是请求的主体。
    //
    // 在客户端，如果Body是nil表示该请求没有主体买入GET请求。
    // Client的Transport字段会负责调用Body的Close方法。
    //
    // 在服务端，Body字段总是非nil的；但在没有主体时，读取Body会立刻返回EOF。
    // Server会关闭请求的主体，ServeHTTP处理器不需要关闭Body字段。
    Body io.ReadCloser
    // ContentLength记录相关内容的长度。
    // 如果为-1，表示长度未知，如果>=0，表示可以从Body字段读取ContentLength字节数据。
    // 在客户端，如果Body非nil而该字段为0，表示不知道Body的长度。
    ContentLength int64
    // TransferEncoding按从最外到最里的顺序列出传输编码，空切片表示"identity"编码。
    // 本字段一般会被忽略。当发送或接受请求时，会自动添加或移除"chunked"传输编码。
    TransferEncoding []string
    // Close在服务端指定是否在回复请求后关闭连接，在客户端指定是否在发送请求后关闭连接。
    Close bool
    // 在服务端，Host指定URL会在其上寻找资源的主机。
    // 根据RFC 2616，该值可以是Host头的值，或者URL自身提供的主机名。
    // Host的格式可以是"host:port"。
    //
    // 在客户端，请求的Host字段（可选地）用来重写请求的Host头。
    // 如过该字段为""，Request.Write方法会使用URL字段的Host。
    Host string
    // Form是解析好的表单数据，包括URL字段的query参数和POST或PUT的表单数据。
    // 本字段只有在调用ParseForm后才有效。在客户端，会忽略请求中的本字段而使用Body替代。
    Form url.Values
    // PostForm是解析好的POST或PUT的表单数据。
    // 本字段只有在调用ParseForm后才有效。在客户端，会忽略请求中的本字段而使用Body替代。
    PostForm url.Values
    // MultipartForm是解析好的多部件表单，包括上传的文件。
    // 本字段只有在调用ParseMultipartForm后才有效。
    // 在客户端，会忽略请求中的本字段而使用Body替代。
    MultipartForm *multipart.Form
    // Trailer指定了会在请求主体之后发送的额外的头域。
    //
    // 在服务端，Trailer字段必须初始化为只有trailer键，所有键都对应nil值。
    // （客户端会声明哪些trailer会发送）
    // 在处理器从Body读取时，不能使用本字段。
    // 在从Body的读取返回EOF后，Trailer字段会被更新完毕并包含非nil的值。
    // （如果客户端发送了这些键值对），此时才可以访问本字段。
    //
    // 在客户端，Trail必须初始化为一个包含将要发送的键值对的映射。（值可以是nil或其终值）
    // ContentLength字段必须是0或-1，以启用"chunked"传输编码发送请求。
    // 在开始发送请求后，Trailer可以在读取请求主体期间被修改，
    // 一旦请求主体返回EOF，调用者就不可再修改Trailer。
    //
    // 很少有HTTP客户端、服务端或代理支持HTTP trailer。
    Trailer Header
    // RemoteAddr允许HTTP服务器和其他软件记录该请求的来源地址，一般用于日志。
    // 本字段不是ReadRequest函数填写的，也没有定义格式。
    // 本包的HTTP服务器会在调用处理器之前设置RemoteAddr为"IP:port"格式的地址。
    // 客户端会忽略请求中的RemoteAddr字段。
    RemoteAddr string
    // RequestURI是被客户端发送到服务端的请求的请求行中未修改的请求URI
    // （参见RFC 2616, Section 5.1）
    // 一般应使用URI字段，在客户端设置请求的本字段会导致错误。
    RequestURI string
    // TLS字段允许HTTP服务器和其他软件记录接收到该请求的TLS连接的信息
    // 本字段不是ReadRequest函数填写的。
    // 对启用了TLS的连接，本包的HTTP服务器会在调用处理器之前设置TLS字段，否则将设TLS为nil。
    // 客户端会忽略请求中的TLS字段。
    TLS *tls.ConnectionState
}
```

* **API**

```go
//NewRequest使用指定的方法、网址和可选的主题创建并返回一个新的*Request。

//如果body参数实现了io.Closer接口，Request返回值的Body 字段会被设置为body，并会被Client类型的Do、
//Post和PostFOrm方法以及Transport.RoundTrip方法关闭。

func NewRequest(method, urlStr string, body io.Reader) (*Request, error)

//ParseForm解析URL中的查询字符串，并将解析结果更新到r.Form字段。
//对于POST或PUT请求，ParseForm还会将body当作表单解析，并将结果既更新到r.PostForm也更新到r.Form。解析结果中，POST或PUT请求主体要优先于URL查询字符串（同名变量，主体的值在查询字符串的值前面）。
//如果请求的主体的大小没有被MaxBytesReader函数设定限制，其大小默认限制为开头10MB。
//ParseMultipartForm会自动调用ParseForm。重复调用本方法是无意义的。

func (r *Request) ParseForm() error

//FormValue返回key为键查询r.Form字段得到结果[]string切片的第一个值。POST和PUT主体中的同名参数优先于URL查询字符串。如果必要，本函数会隐式调用ParseMultipartForm和ParseForm。

func (r *Request) FormValue(key string) string
//FormFile返回以key为键查询r.MultipartForm字段得到结果中的第一个文件和它的信息。
//如果必要，本函数会隐式调用ParseMultipartForm和ParseForm。查询失败会返回ErrMissingFile错误。
func (r *Request) FormFile(key string) (multipart.File, *multipart.FileHeader, error)
```

###### 4.4 ResponseWriter

* **接口**

```
ResponseWriter接口被HTTP处理器用于构造HTTP回复。
type ResponseWriter interface {
    // Header返回一个Header类型值，该值会被WriteHeader方法发送。
    // 在调用WriteHeader或Write方法后再改变该对象是没有意义的。
    Header() Header
    // WriteHeader该方法发送HTTP回复的头域和状态码。
    // 如果没有被显式调用，第一次调用Write时会触发隐式调用WriteHeader(http.StatusOK)
    // WriterHeader的显式调用主要用于发送错误码。
    WriteHeader(int)
    // Write向连接中写入作为HTTP的一部分回复的数据。
    // 如果被调用时还未调用WriteHeader，本方法会先调用WriteHeader(http.StatusOK)
    // 如果Header中没有"Content-Type"键，
    // 本方法会使用包函数DetectContentType检查数据的前512字节，将返回值作为该键的值。
    Write([]byte) (int, error)
}
```

##### 5. error处理

###### 5.1 error定义

> `error`其实一个接口，内置的，我们看下它的定义:
>
> ```go
> // The error built-in interface type is the conventional interface for
> // representing an error condition, with the nil value representing no error.
> type error interface {
> 	Error() string
> }
> ```
>
> 它只有一个方法 `Error`，只要实现了这个方法，就是实现了`error`。现在我们自己定义一个错误试试。
>
> ```go
> type fileError struct {
> }
> 
> func (fe *fileError) Error() string {
> 	return "文件错误"
> }
> ```

###### 5.2 自定义error

> 自定义了一个`fileError`类型，实现了`error`接口。现在测试下看看效果。
>
> ```go
> func main() {
> 	conent, err := openFile()
> 	if err != nil {
> 		fmt.Println(err)
> 	} else {
> 		fmt.Println(string(conent))
> 	}
> }
> 
> //只是模拟一个错误
> func openFile() ([]byte, error) {
> 	return nil, &fileError{}
> }
> ```
>
> 我们运行模拟的代码，可以看到`文件错误`的通知。
>
> 在实际的使用过程中，我们可能遇到很多错误，他们的区别是错误信息不一样，一种做法是每种错误都类似上面一样定义一个错误类型，但是这样太麻烦了。我们发现`Error`返回的其实是个字符串，我们可以修改下，让这个字符串可以设置就可以了。
>
> ```go
> type fileError struct {
> 	s string
> }
> 
> func (fe *fileError) Error() string {
> 	return fe.s
> }
> ```
>
> 恩，这样改造后，我们就可以在声明`fileError`的时候，设置好要提示的错误文字，就可以满足我们不同的需要了。
>
> ```go
> //只是模拟一个错误
> func openFile() ([]byte, error) {
> 	return nil, &fileError{"文件错误，自定义"}
> }
> ```
>
> 恩，可以了，已经达到了我们的目的。现在我们可以把它变的更通用一些，比如修改`fileError`的名字，再创建一个辅助函数，便于我们创建不同的错误类型。
>
> ```go
> func New(text string) error {
> 	return &errorString{text}
> }
> 
> type errorString struct {
> 	s string
> }
> 
> func (e *errorString) Error() string {
> 	return e.s
> }
> ```
>
> 变成以上这样，我们就可以通过`New`函数，辅助我们创建不同的错误了，这其实就是我们经常用到的`errors.New`函数，被我们一步步剖析演化而来，现在大家对Go语言(golang)内置的错误`error`有了一个清晰的认知了。

###### 5.3 存在的问题

> 虽然Go语言对错误的设计非常简洁，但是对于我们开发者来说，很明显是不足的，比如我们需要知道出错的更多信息，在什么文件的，哪一行代码？只有这样我们才更容易的定位问题。
>
> 还有比如，我们想对返回的`error`附加更多的信息后再返回，比如以上的例子，我们怎么做呢？我们只能先通过`Error`方法，取出原来的错误信息，然后自己再拼接，再使用`errors.New`函数生成新错误返回。
>
> 如果我们以前做过java开发，我们知道Java的异常是可以嵌套的，也就是说，通过这个，我们很容易知道错误的根本原因，因为Java的异常，是一层层的嵌套返回的，不管中间经历了多少包装，我们可以通过`cause`找到根本错误的原因。

###### 5.4 解决问题

> 如果要解决以上的问题，那么首先我们必须再继续扩充我们的`errorString`，再增加一些字段来存储更多的信息。比如我们要记录堆栈信息。
>
> ```go
> type stack []uintptr
> type errorString struct {
> 	s string
> 	*stack
> }
> ```
>
> 有了存储堆栈信息的`stack`字段，我们在生成错误的时候，就可以把调用的堆栈信息存储在这个字段里。
>
> ```go
> func callers() *stack {
> 	const depth = 32
> 	var pcs [depth]uintptr
> 	n := runtime.Callers(3, pcs[:])
> 	var st stack = pcs[0:n]
> 	return &st
> }
> 
> func New(text string) error {
> 	return &errorString{
> 		s:   text,
> 		stack: callers(),
> 	}
> }
> ```
>
> 完美解决，现在如果再解决，对现有的错误附加一些信息的问题呢？相信大家应该有思路了。
>
> ```go
> type withMessage struct {
> 	cause error
> 	msg   string
> }
> 
> func WithMessage(err error, message string) error {
> 	if err == nil {
> 		return nil
> 	}
> 	return &withMessage{
> 		cause: err,
> 		msg:   message,
> 	}
> }
> ```
>
> 使用`WithMessage`函数，对原来的`error`包装下，就可以生成一个新的带有包装信息的错误了。

###### 5.5 推荐的方案

> 以上我们在解决问题是，采取的方法是不是比较熟悉？尤其是看源代码，没错，这就是`github.com/pkg/errors`这个错误处理库的源代码。
>
> 因为Go语言提供的错误太简单了，以至于简单的我们无法更好的处理问题，甚至不能为我们处理错误，提供更有用的信息，所以诞生了很多对错误处理的库，`github.com/pkg/errors`是比较简洁的一样，并且功能非常强大，受到了大量开发者的欢迎，使用者很多。
>
> 它的使用非常简单，如果我们要新生成一个错误，可以使用`New`函数,生成的错误，自带调用堆栈信息。
>
> ```go
> func New(message string) error
> ```
>
> 如果有一个现成的`error`，我们需要对他进行再次包装处理，这时候有三个函数可以选择。
>
> ```go
> //只附加新的信息
> func WithMessage(err error, message string) error
> 
> //只附加调用堆栈信息
> func WithStack(err error) error
> 
> //同时附加堆栈和信息
> func Wrap(err error, message string) error
> ```
>
> 其实上面的包装，很类似于Java的异常包装，被包装的`error`，其实就是`Cause`,在前面的章节提到错误的根本原因，就是这个`Cause`。所以这个错误处理库为我们提供了`Cause`函数让我们可以获得最根本的错误原因。
>
> ```go
> func Cause(err error) error {
> 	type causer interface {
> 		Cause() error
> 	}
> 
> 	for err != nil {
> 		cause, ok := err.(causer)
> 		if !ok {
> 			break
> 		}
> 		err = cause.Cause()
> 	}
> 	return err
> }
> ```
>
> 使用`for`循环一直找到最根本（最底层）的那个`error`。
>
> 以上的错误我们都包装好了，也收集好了，那么怎么把他们里面存储的堆栈、错误原因等这些信息打印出来呢？其实，这个错误处理库的错误类型，都实现了`Formatter`接口，我们可以通过`fmt.Printf`函数输出对应的错误信息。
>
> ```go
> %s,%v //功能一样，输出错误信息，不包含堆栈
> %q //输出的错误信息带引号，不包含堆栈
> %+v //输出错误信息和堆栈
> ```
>
> 以上如果有循环包装错误类型的话，会递归的把这些错误都会输出。

###### 5.6 小结

> 通过使用这个 `github.com/pkg/errors` 错误库，我们可以收集更多的信息，可以让我们更容易的定位问题。
>
> 我们收集的这些信息不止可以输出到控制台，也可以当做日志，使用输出到相应的`Log`日志里，便于分析问题。
>
> 据说这个库，会被加入到Golang 标准 SDK 里，期待着，如果加入的话，应该就是补充现在标准库里的`errors` 这个package了。

##### 6. Error Wrapping

Go 1.13发布的功能还有一个值得深入研究的，就是对Error的增强，也是今天我们要分析的 Error Wrapping.

###### 6.1 背景

> 做Go语言开发的，肯定经常用`error`，但是我们也知道`error`非常弱，只能自带一串文本其他什么都做不了，比如给已经存在的`error`增加一些附加文本，增加堆栈信息等都做不了。如果我们想给`error`增加一些附加文本怎么做呢？有两种办法：
>
> 第一种：
>
> ```go
> newErr:=fmt.Errorf("数据上传问题: %v", err)
> ```
>
> 通过`fmt.Errorf`函数，基于已经存在的`err`再生成一个新的`newErr`，然后附加上我们想添加的文本信息。这种办法比较方便，但是问题也很明显，我们丢失了原来的`err`，因为它已经被我们的`fmt.Errorf`函数转成一个新的字符串了。
>
> 第二种：
>
> ```go
> func main() {
> 	newErr := MyError{err, "数据上传问题"}
> }
> 
> type MyError struct {
> 	err error
> 	msg string
> }
> 
> func (e *MyError) Error() string {
> 	return e.err.Error() + e.msg
> }
> ```
>
> 这种方式就是我们自定义自己的`struct`，添加用于存储我们需要额外信息的字段，我这里是`err`和`msg`，分别存储原始`err`和新附加的出错信息。然后这个`MyError`还会实现`error`接口，表示他是一个`error`，这样我们就可以自由的使用了。
>
> 这种方式有点很明显，大家可以看到了。缺点呢，就是我们要自定义很多`struct`，而且我们自己定义的，和第三方的可能还不太一样，无法统一和兼容。基于这个背景，Golang 1.13 为我们提供了Error Wrapping，翻译过来我更愿意叫Error嵌套。

###### 6.2 如何生成一个Error Wrapping

> Error Wrapping，顾名思义，就是为我们提供了，可以一个`error`嵌套另一个`error`功能，好处就是我们可以根据嵌套的`error`序列，生成一个`error`错误跟踪链，也可以理解为错误堆栈信息，这样可以便于我们跟踪调试，哪些错误引起了什么问题，根本的问题原因在哪里。
>
> 因为`error`可以嵌套，所以每次嵌套的时候，我们都可以提供新的错误信息，并且保留原来的`error`。现在我们看下如何生成一个嵌套的`error`。
>
> ```go
> e := errors.New("原始错误e")
> w := fmt.Errorf("Wrap了一个错误%w", e)
> ```
>
> Golang并没有提供什么`Wrap`函数，而是扩展了`fmt.Errorf`函数，加了一个`%w`来生成一个可以Wrapping Error,通过这种方式，我们可以创建一个个以Wrapping Error。

###### 6.3 Error Wrapping原理

> 按照这种不丢失原`error`的思路，那么Wrapping Error的实现原理应该类似我们上面的自定义`error`.我们看下`fmt.Errorf`函数的源代码验证下我们的猜测是否正确。
>
> ```go
> func Errorf(format string, a ...interface{}) error {
> 	//省略无关代码
> 	var err error
> 	if p.wrappedErr == nil {
> 		err = errors.New(s)
> 	} else {
> 		err = &wrapError{s, p.wrappedErr}
> 	}
> 	p.free()
> 	return err
> }
> ```
>
> 这里的关键核心代码就是`p.wrappedErr`的判断，这个值是否存在，决定是否要生成一个wrapping error。这个值是怎么来的呢？就是根据我们设置的`%w`解析出来的。
>
> 有了这个值之后，就生成了一个`&wrapError{s, p.wrappedErr}`返回了，这里有个结构体`wrapError`
>
> ```go
> type wrapError struct {
> 	msg string
> 	err error
> }
> 
> func (e *wrapError) Error() string {
> 	return e.msg
> }
> 
> func (e *wrapError) Unwrap() error {
> 	return e.err
> }
> ```
>
> 如上所示，和我们想的一样。实现了`Error`方法说明它是一个`error`。`Unwrap`方法是一个特别的方法，所有的wrapping error 都会有这么一个方法，用于获得被嵌套的error。

###### 6.4 Unwrap函数

> Golang 1.13引入了wrapping error后，同时为`errors`包添加了3个工具函数，他们分别是`Unwrap`、`Is`和`As`，先来聊聊`Unwrap`。
>
> 顾名思义，它的功能就是为了获得被嵌套的error。
>
> ```go
> func main() {
> 	e := errors.New("原始错误e")
> 	w := fmt.Errorf("Wrap了一个错误%w", e)
> 	fmt.Println(errors.Unwrap(w))
> }
> ```
>
> 以上这个例子，通过`errors.Unwrap(w)`后，返回的其实是个`e`，也就是被嵌套的那个error。 这里需要注意的是，嵌套可以有很多层，我们调用一次`errors.Unwrap`函数只能返回最外面的一层`error`，如果想获取更里面的，需要调用多次`errors.Unwrap`函数。最终如果一个`error`不是warpping error，那么返回的是`nil`。
>
> ```go
> func Unwrap(err error) error {
>     //先判断是否是wrapping error
>     //注意此处的判断手法
> 	u, ok := err.(interface {
> 		Unwrap() error
> 	})
> 	//如果不是，返回nil
> 	if !ok {
> 		return nil
> 	}
> 	//否则则调用该error的Unwrap方法返回被嵌套的error
> 	return u.Unwrap()
> }
> ```
>
> 看看该函数的的源代码吧，这样就会理解的更深入一些，我加了一些注释。

###### 6.5 Is函数

> 在Go 1.13之前没有wrapping error的时候，我们要判断error是不是同一个error可以使用如下办法：
>
> ```go
> if err == os.ErrExist
> ```
>
> 这样我们就可以通过判断来做一些事情。但是现在有了wrapping error后这样办法就不完美的，因为你根本不知道返回的这个`err`是不是一个嵌套的error,嵌套了几层。所以基于这种情况，Golang为我们提供了`errors.Is`函数。
>
> ```
> func Is(err, target error) bool
> ```
>
> 1. 如果`err`和`target`是同一个，那么返回`true`
> 2. 如果`err` 是一个wrap error,`target`也包含在这个嵌套error链中的话，那么也返回`true`。
>
> 很简单的一个函数，要么咱俩相等，要么`err`包含`target`，这两种情况都返回`true`，其余返回`false`。
>
> ```go
> func Is(err, target error) bool {
> 	if target == nil {
> 		return err == target
> 	}
> 
> 	isComparable := reflectlite.TypeOf(target).Comparable()
> 	
> 	//for循环，把err一层层剥开，一个个比较，找到就返回true
> 	for {
> 		if isComparable && err == target {
> 			return true
> 		}
> 		//这里意味着你可以自定义error的Is方法，实现自己的比较代码
> 		if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
> 			return true
> 		}
> 		//剥开一层，返回被嵌套的err
> 		if err = Unwrap(err); err == nil {
> 			return false
> 		}
> 	}
> }
> ```
>
> `Is`函数源代码如上，其实就是一层层反嵌套，剥开然后一个个的和`target`比较，相等就返回true。

###### 6.6 As函数

> 在Go 1.13之前没有wrapping error的时候，我们要把error转为另外一个error，一般都是使用type assertion 或者 type switch，其实也就是类型断言。
>
> ```go
> if perr, ok := err.(*os.PathError); ok {
> 	fmt.Println(perr.Path)
> }
> ```
>
> 比如例子中的这种方式，但是现在给你返回的err可能是已经被嵌套了，甚至好几层了，这种方式就不能用了，所以Golang为我们在`errors`包里提供了`As`函数，现在我们把上面的例子，用`As`函数实现一下。
>
> ```go
> var perr *os.PathError
> if errors.As(err, &perr) {
> 	fmt.Println(perr.Path)
> }
> ```
>
> 这样就可以了，就可以完全实现类型断言的功能，而且还更强大，因为它可以处理wrapping error。
>
> ```go
> func As(err error, target interface{}) bool
> ```
>
> 从功能上来看，`As`所做的就是遍历err嵌套链，从里面找到类型符合的error，然后把这个error赋予target,这样我们就可以使用转换后的target了，这里有值得赋予，所以target必须是一个指针。
>
> ```go
> func As(err error, target interface{}) bool {
>     //一些判断，保证target，这里是不能为nil
> 	if target == nil {
> 		panic("errors: target cannot be nil")
> 	}
> 	val := reflectlite.ValueOf(target)
> 	typ := val.Type()
> 	
> 	//这里确保target必须是一个非nil指针
> 	if typ.Kind() != reflectlite.Ptr || val.IsNil() {
> 		panic("errors: target must be a non-nil pointer")
> 	}
> 	
> 	//这里确保target是一个接口或者实现了error接口
> 	if e := typ.Elem(); e.Kind() != reflectlite.Interface && !e.Implements(errorType) {
> 		panic("errors: *target must be interface or implement error")
> 	}
> 	targetType := typ.Elem()
> 	for err != nil {
> 	    //关键部分，反射判断是否可被赋予，如果可以就赋值并且返回true
> 	    //本质上，就是类型断言，这是反射的写法
> 		if reflectlite.TypeOf(err).AssignableTo(targetType) {
> 			val.Elem().Set(reflectlite.ValueOf(err))
> 			return true
> 		}
> 		//这里意味着你可以自定义error的As方法，实现自己的类型断言代码
> 		if x, ok := err.(interface{ As(interface{}) bool }); ok && x.As(target) {
> 			return true
> 		}
> 		//这里是遍历error链的关键，不停的Unwrap，一层层的获取err
> 		err = Unwrap(err)
> 	}
> 	return false
> }
> ```
>
> 这是`As`函数的源代码，看源代码比较清晰一些，我在代码里做了注释，这里就不一一分析了，大家可以结合注释读一下。

###### 6.7 旧工程迁移

> 新特性的更新，如果要使想使用，不免会有旧项目的迁移，现在我们就针对几种常见的情况看如何进行迁移。 如果你以前是直接返回err，或者通过如下方式给err增加了额外信息。
>
> ```
> return err
> 
> return fmt.Errorf("more info: %v", err)
> ```
>
> 这2种情况你直接切换即可。
>
> ```go
> return fmt.Errorf("more info: %w", err)
> ```
>
> 切换后，如果你有`==`的error判断，那么就用`Is`函数代替，比如：
>
> *旧工程*
>
> ```go
> if err == os.ErrExist
> ```
>
> *新工程*
>
> ```go
> if errors.Is(err, os.ErrExist)
> ```
>
> 同理，你旧的代码中，如果有对error进行类型断言的转换，就要用`As`函数代替，比如：
>
> *旧工程*
>
> ```go
> if perr, ok := err.(*os.PathError); ok {
> 	fmt.Println(perr.Path)
> }
> ```
>
> *新工程*
>
> ```go
> var perr *os.PathError
> if errors.As(err, &perr) {
> 	fmt.Println(perr.Path)
> }
> ```
>
> 如果你自己自定义了一个`struct`实现了`error`接口，而且还嵌套了`error`，这个时候该怎么适配新特性呢？也就是我们上面举例的情况：
>
> ```go
> type MyError struct {
> 	err error
> 	msg string
> }
> 
> func (e *MyError) Error() string {
> 	return e.err.Error() + e.msg
> }
> ```
>
> 其实对于这种方式很简单，只需要给他添加一个`Unwrap`方法就可以了，让它变成一个wrap error。
>
> ```go
> func (e *MyError) Unwrap() error {
> 	return e.err
> }
> ```
>
> 这样就可以了。



##### 7. time

###### 7.1 相关API

> * 获取当前时间
>
> ```go
> import "time"
> //func Now() Time
> //获取当前时间
> now := time.Now()
> fmt.Printf("now is %v  and type is %t \n",now,now)
> 
> ```
>
> ![image-20200325104252289](C:\Users\Hunter-T430\AppData\Roaming\Typora\typora-user-images\image-20200325104252289.png)
>
> * 获取时间的其他信息
>
> ```go
> import (
> 	"time"
>     "fmt"
> )
> //func Now() Time
> //获取当前时间
> now := time.Now()
> fmt.Printf("now is %v  and type is %t \n",now,now)
> 
> fmt.Printf("年： %v \n",now.Year())
> fmt.Printf("月： %v \n",int(now.Month()))
> fmt.Printf("日： %v \n",now.Day())
> fmt.Printf("时： %v \n",now.Hour())
> fmt.Printf("分： %v \n",now.Minute())
> fmt.Printf("秒： %v \n",now.Second())
> ```
>
> * 时间的格式化输出
>
> ```go
> //使用fmt.Sprintf()
> datestr := fmt.Sprintf("当前时间是%d-%d-%d %d:%d:%d \n",now.Year(),now.Month(),now.Day(),now.Hour(),now.Minute().now.Second())
> fmt.Printf("时间是%v， \n",datestr)
> ```
>
> 

##### 8. flag

###### 8.1 flag包简介

> go语言内置的flag包实现了命令行参数的解析，
>
> 即当使用命令行启动一个程序时可以通过命令行向程序中添加一些参数
>
> flag包使得开发命令行工具更为简单

###### 8.2 os.Args()

> 如果只是简单的获取命令行参数，可以像下面的示例代码一样使用os.Args来获取命令行参数
>
> ```go
> package main
> import "fmt"
> import "os"
> func main(){
>     if len(os.Args)>0 {
>         for index , value := range os.Args {
>             fmt.Println(index, value)
>         }
>     }
> }
> ```
>
> 将上面的代码执行go build main.go编译之后，执行：
>
> ```go
> 0 ./main
> //输出
> 1 a
> 2 b
> 3 c
> 4 d
> ```
>
> os.Args是一个存储命令行参数的字符串切片，它的第一个元素是执行文件的名称。

###### 8.3 flag的使用

> flag包支持的命令行参数类型有bool、int、int64、uint、uint64、float、float64、string、duration.
>
> * #### flag.Type()
>
>   基本格式如下：
>   flag.Type(flag名，默认值，帮助信息) *Type 
>
>   例如我们要定义姓名、年龄、婚否三个命令行参数，我们可以按照如下定义：
>
>   ```go
>    name := flag.String("name","ali","姓名")
>    age := flag.Int("age",18,"年龄")
>    married := flag.Bool("married",false,"婚否")
>    delay := flag.Duration("d",0,"时间间隔")
>   ```
>
>   需要注意的是，此时的 name,age,married,delay均为对应类型的指针。
>
> * #### flag.TypeVar()
>
>   基本格式如下：flag.TypeVar(Type指针，flag名，默认值，帮助信息)
>
>   例如我们要定义姓名、年龄、婚否三个命令行参数，
>
>   我们可以按照如下方式定义：
>
>   ```go
>   var name string
>   var age int
>   var married bool
>   var delay time.Duration
>   flag.StringVar(&name,"name","张三","姓名")
>   flag.IntVar(&age,"age",18,"年龄")
>   flag.BoolVar(&married,"married",false,"婚否")
>   flag.Duration(&delay,"d",0,"时间间隔")
>   ```
>
> * #### flag.Parse()
>
>   通过以上两种方法定义命令行flag参数后，需要通过调用flag.Parse()来对命令行参数进行解析。
>   支持的命令行参数格式有一下几种：
>
>   - -flag xxx (使用空格，一个 - 符号）
>   - --flag xxx (使用空格，两个 - 符号)
>   - -flag=xxx （使用等号， 一个 - 符号）
>   - --flag = xxx (使用等号， 两个- 符号)
>
>   其中，布尔类型的参数必须用等号的方式指定。
>   flag在解析第一个非flag参数之前停止，或者在终止符"-"之后停止
>
> * #### flag其他函数
>
>   ```go
>   flag.Args()  //返回命令行参数后的其他参数，以[]string类型
>   flag.NArg() //返回命令行参数后的其他参数个数
>   flag.NFlag() //返回使用命令行参数个数
>   ```

###### 8.4 完整示例

```go
package main
import (
    "fmt"
    "flag"
    "time"
)
func main(){
    var name string
    var age int
    var married bool
    var delay time.Duration
    flag.StringVar(&name,"name","张三","姓名")
    flag.IntVar(&age,"age",18,"年龄")
    flag.BoolVar(&married,"married",false,"婚否")
    flag.DurationVar(&delay, "d", 0, "延迟的时间间隔")
    flag.Parse()
    fmt.Println(name,age,married,delay)
    fmt.Println(flag.Args())
    fmt.Println(flag.NArg())
    fmt.Println(flag.NFlag())

}
```

* 正常使用命令行flag参数:

  ```go
   ./args_demo --name 王浩 --age 18 --married=false -d 1h30m
  霍帅兵 18 false 1h30m0s
  []
  0
  4
  ```

* 使用非flag命令行参数：

  ```go
  ./args_demo a b c
  张三 18 false 0s
  [a b c]
  3
  0
  ```

  

#### 三 库的使用

##### 1. 二维码的生成

###### 1.1 什么是二维码

> 二维条码是指在一维条码的基础上扩展出另一维具有可读性的条码，使用黑白矩形图案表示二进制数据，被设备扫描后可获取其中所包含的信息。一维条码的宽度记载着数据，而其长度没有记载数据。二维条码的长度、宽度均记载着数据。二维条码有一维条码没有的“定位点”和“容错机制”。容错机制在即使没有辨识到全部的条码、或是说条码有污损时，也可以正确地还原条码上的信息。

###### 1.2 Go生成二维码图片

> 使用Go语言编程时，生成任意内容的二维码是非常方便的，因为我们有`go-qrcode`这个库。该库的源代码托管在github上，大家可以下载使用 https://github.com/skip2/go-qrcode。
>
> 这个库的使用很简单，假如我要以我的个人信息生成一张256*256的图片，可以使用如下代码：
>
> ```go
> import "github.com/skip2/go-qrcode"
> 
> func main() {
> 	qrcode.WriteFile("{"name":"南山一小白","age":17}",qrcode.Medium,256,"./blog_qrcode.png")
> }
> ```
>
> 这样我们运行代码的时候，就在当前目录下，生成一张256*256的二维码，扫描后可以看到内容是`{"name":"南山一小白","age":17}`。
>
> ```go
> func WriteFile(content string, level RecoveryLevel, size int, filename string) error
> ```
>
> `WriteFile`函数的原型定义如上，它有几个参数，大概意思如下：
>
> 1. `content`表示要生成二维码的内容，可以是任意字符串。
> 2. `level`表示二维码的容错级别，取值有Low、Medium、High、Highest。
> 3. `size`表示生成图片的width和height，像素单位。
> 4. `filename`表示生成的文件名路径。
>
> `RecoveryLevel`类型其实是个`int`,它的定义和常量如下。
>
> ```go
> type RecoveryLevel int
> 
> const (
> 	// Level L: 7% error recovery.
> 	Low RecoveryLevel = iota
> 
> 	// Level M: 15% error recovery. Good default choice.
> 	Medium
> 
> 	// Level Q: 25% error recovery.
> 	High
> 
> 	// Level H: 30% error recovery.
> 	Highest
> )
> ```
>
> `RecoveryLevel`越高，二维码的容错能力越好。

###### 1.3 生成二维码图片字节

> 有时候我们不想直接生成一个PNG文件存储，我们想对PNG图片做一些处理，比如缩放了，旋转了，或者网络传输了等，基于此，我们可以使用`Encode`函数，生成一个PNG 图片的字节流，这样我们就可以进行各种处理了。
>
> ```go
> func Encode(content string, level RecoveryLevel, size int) ([]byte, error)
> ```
>
> 用法和`WriteFile`函数差不多，只不过返回的是一个`[]byte`字节数组，这样我们就可以对这个字节数组进行处理了。

###### 1.4 自定义二维码

> 除了以上两种快捷方式，该库还为我们提供了对二维码的自定义方式，比如我们可以自定义二维码的前景色和背景色等。`qrcode.New`函数可以返回一个`*QRCode`，我们可以对`*QRCode`设置，实现对二维码的自定义。
>
> 比如我们设置背景色为绿色，前景色为白色的二维码
>
> ```go
> func main() {
> 	qr,err:=qrcode.New("http://www.flysnow.org/",qrcode.Medium)
> 	if err != nil {
> 		log.Fatal(err)
> 	} else {
> 		qr.BackgroundColor = color.RGBA{50,205,50,255}
> 		qr.ForegroundColor = color.White
> 		qr.WriteFile(256,"./blog_qrcode.png")
> 	}
> }
> ```
>
> 指定`*QRCode`的`BackgroundColor`和`ForegroundColor`即可。然后调用`WriteFile`方法生成这个二维码文件。
>
> ```go
> func New(content string, level RecoveryLevel) (*QRCode, error)
> 
> // A QRCode represents a valid encoded QRCode.
> type QRCode struct {
> 	// Original content encoded.
> 	Content string
> 
> 	// QR Code type.
> 	Level         RecoveryLevel
> 	VersionNumber int
> 
> 	// User settable drawing options.
> 	ForegroundColor color.Color
> 	BackgroundColor color.Color
> }
> ```
>
> 以上`QRCode`的这些字段都是可以设置的，这样我们就可以灵活自定义二维码了。

###### 1.5 小结

> 二维码是一种流行的输入技术手段，不光Go可以生成，其他语言也可以生成，并且生成的二维码是标准的，都可以扫描和识别，比如Java可以通过这个https://github.com/kenglxn/QRGen库来生成。

|      | 星期一                    | 星期二                         | 星期三                                                       | 星期四                         |
| ---- | ------------------------- | ------------------------------ | ------------------------------------------------------------ | ------------------------------ |
| 1    |                           | 计算机操作系统（5-16）教西B413 | 课程:计算智能 班级:1班 主讲教师：罗建超 教室:长安-教学西楼B座-B211(第9-18周 连续周 星期三 上1,上2,上3,上4) | 计算机操作系统（5-16）教西B413 |
| 2    |                           | 计算机操作系统（5-16）教西B413 | 课程:计算智能 班级:1班 主讲教师：罗建超 教室:长安-教学西楼B座-B211(第9-18周 连续周 星期三 上1,上2,上3,上4) | 计算机操作系统（5-16）教西B413 |
| 3    | 数据结构（5-18）教东JB208 |                                | 数据结构（5-18）教东JB208                                    |                                |
| 4    | 数据结构（5-18）教东JB208 |                                | 数据结构（5-18）教东JB208                                    |                                |
|      |                           |                                |                                                              |                                |
|      |                           |                                |                                                              |                                |
| 5    |                           | 随机过程(1-10）教东D座-JD207   |                                                              | 随机过程(1-10）教东D座-JD207   |
| 6    |                           | 随机过程(1-10）教东D座-JD207   |                                                              | 随机过程(1-10）教东D座-JD207   |

##### 2. groupCache 的使用

###### 2.1 简介

> 分布式缓存groupcache:
>
> - 只是一个代码包，可以直接使用，不需要单独配置服务器，既是客户端也是服务器端，这主要是因为它既可能直接返回请求的key的结果，也可能将请求转给其他peer来处理。对访问的用户来说，它是服务端，对其他peer来说它又是客户端（稍微有点绕，不明白的看之后的demo吧）；
> - 分布式缓存，支持一致性哈希，这里主要是通过peers和consistenthash实现的；
> - 采用的缓存机制是LRU，LRU主要通过list.List队列来实现的，不支持过期机制；（所以。。是的，如果有长期不用的数据，只能寄希望于缓存满了之后自动剔除队尾了）
> - 对于客户端或者使用用户来说，对数据的操作只能get，不支持set，update以及delete；
> - 实现了缓存过滤机制(这个概念也是笔者网上看到的)， 代码中叫singlefilght；一般情况下，缓存存在被击穿的风险，即大量相同的请求并发访问，如果此时cache miss，那么这些请求都会落地到服务器端的db上，导致db压力过大甚至最终造成整个服务不可用的情况。而singleflight所做的就是针对并发访问的相同miss key，只有一个请求会真正load data（如访问db），其余请求都会等待，最终将结果返回给其他请求，使得所有请求能返回一致的结果。
> - 对热点数据进行备份，虽然是一致性哈希，但是对于过热数据，也会进行多节点备份，缓解过热数据访问对节点造成的压力；

###### 2.2 代码结构

> 它的代码结构也比较清晰，代码量也不是很大，很适合大家去阅读学习。
>
> 主要分为了
>
> * consistenthash(提供一致性哈希算法的支持)，
> * lru(提供了LRU方式清楚缓存的算法)，
> * singleflight(保证了多次相同请求只去获取值一次，减少了资源消耗)，
>
> 还有一些源文件：
>
> * byteview.go 提供类似于一个数据的容器，
> * http.go提供不同地址间的缓存的沟通的实现，
> * peers.go节点的定义，
> * sinks.go感觉就是一个开辟空间给容器，并和容器交互的一个中间人，
> * groupcache.go整个源码里的大当家，其它人都是为它服务的。

###### 2.3 使用案例

> 在使用这个缓存的时候比较重要的几方面，也是我之前犯错的几个地方。
>
> 需要监听两个地方，
>
> * 一个是监听节点，
>
> * 一个是监听请求
>
> 在批量设置节点地址的时候，需要在地址前面加上http://，因为一开始我没有加上去，所以缓存信息一直不能再节点之间交互。启动的节点地址要与设置的节点地址一致：数量和地址值。
>
> ```go
> 
> package main
> import (
>     "flag"
>     "fmt"
>     "github.com/golang/groupcache"
>     "io/ioutil"
>     "log"
>     "net/http"
>     "os"
>     "strconv"
>     "strings"
> 
> )
> 
>         
> 
> var (
>     // peers_addrs = []string{"127.0.0.1:8001", "127.0.0.1:8002", "127.0.0.1:8003"}
>     //rpc_addrs = []string{"127.0.0.1:9001", "127.0.0.1:9002", "127.0.0.1:9003"}
>     index = flag.Int("index", 0, "peer index")
> 
> )
> 
> func main() {
> 
>     flag.Parse()
>     peers_addrs := make([]string, 3)
>     rpc_addrs := make([]string, 3)
>     if len(os.Args) > 0 {
>         for i := 1; i < 4; i++ {
>             peers_addrs[i-1] = os.Args[i]
>             rpcaddr := strings.Split(os.Args[i], ":")[1]
>             port, _ := strconv.Atoi(rpcaddr)
>             rpc_addrs[i-1] = ":" + strconv.Itoa(port+1000)
>         }
>     }
> 
>     if *index < 0 || *index >= len(peers_addrs) {
>         fmt.Printf("peer_index %d not invalid\n", *index)
>         os.Exit(1)
>     }
> 
>     peers := groupcache.NewHTTPPool(addrToURL(peers_addrs[*index]))
> 
>     var stringcache = groupcache.NewGroup("SlowDBCache", 64<<20, groupcache.GetterFunc(
>         func(ctx groupcache.Context, key string, dest groupcache.Sink) error {
> 
>             result, err := ioutil.ReadFile(key)
>             if err != nil {
>                 log.Fatal(err)
>                 return err
>             }
>             fmt.Printf("asking for %s from dbserver\n", key)
>             dest.SetBytes([]byte(result))
>             return nil
> 
>         }))
> 
>     peers.Set(addrsToURLs(peers_addrs)...)
> 
>     http.HandleFunc("/zk", func(rw http.ResponseWriter, r *http.Request) {
> 
>         log.Println(r.URL.Query().Get("key"))
>         var data []byte
>         k := r.URL.Query().Get("key")
>         fmt.Printf("cli asked for %s from groupcache\n", k)
>         stringcache.Get(nil, k, groupcache.AllocatingByteSliceSink(&data))
>         rw.Write([]byte(data))
>         
>     })
>     
>     go http.ListenAndServe(rpc_addrs[*index], nil)
>     rpcaddr := strings.Split(os.Args[1], ":")[1]
>     log.Fatal(http.ListenAndServe(":"+rpcaddr, peers))
>     
> }
> 
> 
> 
> func addrToURL(addr string) string {
>     return "http://" + addr
> }
> 
> 
> 
> func addrsToURLs(addrs []string) []string {
>    
>     result := make([]string, 0)
>     for _, addr := range addrs {
>         result = append(result, addrToURL(addr))
>     }
>     return result
> }
> 
> ```
>
> * 执行方式
>
>   ```go
>   ./demo 127.0.0.1:8001 127.0.0.1:8002 127.0.0.1:8003
>   ```
>
>   在上面的命令中我们就启动了一个节点，并且设置节点地址个数为3个，这里由于我是默认的index为0，所以在启动其它节点的时候变换下地址的顺序，使第一个地址三次都不一样就好了。
>
> * 访问
>
>   打开浏览器访问127.0.0.1:9001?key=1.txt，这里1.txt是需要获取数据的之际地方，类似于实际中的数据库，我这里直接用一个文件代替了
>
> * 运行结果
>
>   当我访问:9001时
>
>   ![img](http://betazk.github.io/image/2014blog/20141203_0.PNG)
>
>   
>
>   ![img](http://betazk.github.io/image/2014blog/20141203_1.PNG)
>
>   
>
>   ![img](http://betazk.github.io/image/2014blog/20141203_2.PNG)
>
>   结果在上面图中，我们像:8001这个节点请求数据，它没有缓存数据，然后会去其它节点中寻找这个key的数据，当都没有的时候就会去数据库中获取(这里我们是一个文件),所以会出现在:8003这个节点中获取数据库数据的操作。
>
>   然后我访问:9002
>
>   ![img](http://betazk.github.io/image/2014blog/20141203_0.PNG)
>
>   
>
>   ![img](http://betazk.github.io/image/2014blog/20141203_3.PNG)
>
>   
>
>   ![img](http://betazk.github.io/image/2014blog/20141203_2.PNG)
>
>   根据上图看到，第一个地址为:8002这个节点直接从缓存里面取值，但是在请求之前这个节点并没有缓存数据，这个也同样是节点间的交互完成的。
>   
>
>   
>
>   

##### 1. cache2go

> https://github.com/muesli/cache2go
>
> 比较简单的一个缓存库，代码量很少，适合新手学习，可以学习到锁、goroutines等。

#### 五. 细节总结

##### 1. 浅拷贝

> slice,map 等赋值都是浅拷贝，并未实现clone

```go
type Test struct{
	A string
	A []string
} 

func main() {
    x := Test{"x-A",[]string{"x-B"}}
    
    y := x 	//copy
    
    y.A = "y-A"
    y.B[0] = "y-B"
    
    fmt.Println(x,y)
    //outPut: "{x-A,[y-b]} {y-A,[y-B]}"
    //此处结构体x中的切片也伴随着y拷贝中的修改而发生变化
}
```

##### 2. Append的使用

> 通过Append修改slice,在cap不够的时候，会出现类似“copy on write"的奇效

```go
func doStuff(value []string){
    fmt.Printf("value= %v \n",value)
    
    //注意此处[:]实际上期望value2和value是同一个slice		存疑
    //执行 value2 := value[:]，两个slice都是指向同一个底层数组，当value2进行append时，原有的底层数组容量不足以容纳新的元素，所以未value2分配了一个更大的底层数组，这样后面修改value2[0]不会影响到value[0].但是第二次调用时，底层数组容量足够大，不会分配新的数组，value2[0]和value[0]是同一个元素。
    value2 := value[:]	//拷贝slice
    value2 := append(value2,"b")
    fmt.Printf("value=%v,value2=%v \n",value,value2)
    
    value2[0] = "2"
    fmt.Printf("value=%v,value2=%v \n",value,value2)
}

func main(){
    slice1 := []string{"a"}
    
    doStuff(slice1)
    //output 
    //value=[a]
    //value=[a],value=[a,b]
    //value=[a],value=[2,b]
    
    slice10 := make([]string,1,10)
    slice10[0] = "a"
    doStuff(slice10)
    //output 
    //value=[a]
    //value=[a],value=[a,b]
    //此处值发生了变化
    //value=[2],value=[2,b]
}
```

