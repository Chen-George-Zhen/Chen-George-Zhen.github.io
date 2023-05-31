---
title: 《Ruby 元编程》总结
author: George
date: 2019-08-09 20:55:00 +0800
categories: [Blogging, Ruby Meta Programming]
tags: [Ruby]
pin: true
---

## 打开类

### Class关键字
可以重新打开已经存在的类并对之进行动态修改，即使像String或者Array这样标准库的类也不例外。这种行为方式称之为打开类(open class)
Ruby的class关键字更像是一个作用域操作符，而不是类型声明语句。class关键字的核心任务是把你带到类的上下文中，让你可以在里面定义方法。如果类不存在，则会起到创建的作用
```ruby
  class D
    def x; 'x'; end
  end
  class D
    def y; 'y'; end
  end
  object = D.new
  object.x # => x
  object.y # => y
```

### 猴子补丁
打开类引起的问题，方法重复定义会覆盖之前的方法
如果你粗心地为某个类添加了新功能，同时覆盖了类原来的功能，进而影响到其他部分的代码，这样的patch称之为猴子补丁(Monkeypatch)

## 类与模块

Ruby的class关键字更像是一个作用域操作符，而不是类型声明语句。class关键字的核心任务是把你带到类的上下文中，让你可以在里面定义方法。

每个类都是一个模块，类就是带有三个方法（new，allocate，superclass）的增强模块，通过这三个方法可以组织类的继承结构，并创建对象
```ruby
  Class.superclass # => Module
  # false 会忽略继承
  Class.instance_methods(false) # => [:allocate, :superclass, :new]
```
Ruby中的类和模块的概念十分接近，完全可以将二者相互替代，之所以同时保留二者的原因是为了保持代码的清晰性，让代码意图更加明确。使用原则:
> 希望把自己代码包含(include)到别的代码中，应该使用模块
> 希望某段代码被实例化或被继承，应该使用类

模块机制可以用来实现类似其它语言中的命名空间(Namespace)概念

### 常量
Ruby中常量的路径(作用域)，类似与文件系统中的目录，通过::进行分割和访问，默认直接以::开头(例: :: Y)表示变量路径的根位置
```ruby
  module M
    class C
      X = "a constant"
    end
    C::X # => "a constant"
  end
  M::C::X # => "a constant"
```

### 什么是对象

对象就是一组实例变量外加一个指向其类的引用。对象的方法并不存在于对象本身，而是存在于对象的类中。
```ruby
  "string".class # => String
  "string".instance_variables # => String
  "string".methods # "string" 对象的方法
  String.instance_methods # String类的实例方法
  "string".methods - String.instance_methods # => []
  String.class # => Class
```

### 什么是类

类就是一个对象(Class类的一个实例)外加一组实例方法和一个对其超类的引用。Class类是Module类的子类，因此一个类也是一个模块。
Ruby中Class类也是对象,也就是Class类的一个实例。在理解和和表述的时候要将 ‘类’ 和 ‘Class类’ 区分开。
```ruby
  # 元类 (单件类)
  String.singleton_class # => #<Class:String>
  # 是类作为对象看待时的类，所以元类的实例方法，就是类的类方法
  string_meta_class = class << String
    #元类实例方法
    def custom_str
      "custom_str"
    end
    # 返回元类
    self
  end
  String.class # => Class
  string_meta_class # => #<Class:String>
  string_meta_class.instance_methods.grep(/custom_str/) # => [:custom_str]
  String.instance_methods.grep(/custom_str/) # => []
  String.singleton_class.instance_methods.grep(/custom_str/) # => [:custom_str]
  String.custom_str #=> "custom_str"

  String.instance_methods
  String.superclass # => Object

  Class.superclass # => Moudule
  Class.class # => Class
```

### load与require方法的异同

通过load和require都可以进行导入别人的代码，不同的是load方法用来加载代码，如果不希望污染当前的命名空间，需要通过load(‘file.rb’,true)显式的要求创建一个匿名模块来，接管file.rb的常量，require用于导入类库，此外，就加载次数上load方法每次调用都会再次运行所加载文件，require则对每个库文件只加载一次。

### prepend、include与祖先链

祖先链用于描述Ruby对象的继承关系，因为类与模块是父子关系，所以祖先链中也可以包含模块，prepend与include分别可以向链中添加模块，不同的是调用include方法，模块会被插入祖先链，当前类的正上方，而prepend同样是插入到祖先链，但位置其他却在当前类的正下方,另外通过Class.ancestors可以查看当前的祖先链

### 方法查找
‘向右一步，在向上’，先查模块，再查祖先链

### private规则

不能通过明确指定接受者来调用私有方法。私有方法只能通过隐性的接受者self调用（Object#send是个例外）

### self相关

调用一个方法时，接受者会扮演self角色
任何没有明确指定接受者的方法调用，都当做是调用self的方法
定义一个模块(或类)时，该模块(或类)会扮演self角色

### 对象、类与模块之间关系

上面Module.class指向的也是Class类，可以理解为上面方框内容均为Class，但他们的父子组织关系通过superclass建立并存在异同，可以通过Class.ancestors查看。

## 动态方法

### 动态调用方法

在Ruby中通过Object#send方法可以代替点标识调用对象的指定实例方法

### 示例代码

```ruby
class MyClass
    def my_method(my_arg)
        my_arg * 2
    end
end

obj = MyClass.new
obj.my_method(3)    #=> 6
obj.send(:my_method, 3) #=> 6
```

上面代码通过直接调用和使用send方法调用得到的结果是一样的，使用send的好处是，可以在编码中，动态的决定方法调用。这个技巧在元编程中被称为动态派发

另外需要指出的地方是通过Object#send不仅可以调用公共方法，也可以调用对象的私有方法。如果想保留对象的封装特性，不向外暴露私有方法可以使用Object#public_send方法。

## 动态定义方法

除了方法的动态调用之外，Ruby还通过Module#define_method方法和代码块提供了动态方法定义方式

### 示例代码
```ruby
class MyClass
    define_method :my_method do |my_arg|
        my_arg * 3
    do
end

obj = MyClass.new
obj.my_method(2)  #=> 6
```

上面代码通过define_method方法取代了关键词def，其本质上都是相同的，只是在定义方式上，define_method的方式更加灵活一些，可以通过在编码中通过推导，完成函数的定义，增加了实现的灵活性。

## method_missing方法

严格意义上将method_missing方法，并不算是明确的定义(不会出现在methods列表中)，其本质是通过方法查找的机制来截获调用信息进而合理的给出相应方法的回应。有点类似与异常处理中的抛出异常，一层一层的往外抛。

method_missing利用的机制是，当一个对象进行某个方法调用的时候，会到其对应的类的实例方法中进行查找，如果没有找到，则顺着祖先链向上查找，直到找到BasicObject类为止。如果都没有则会最终调用一个BasicObject#method_missing抛出NoMethodError异常。

当我们需要定义很多相似的方法时候，可以通过重写method_missing方法,对相似的方法进行统一做出回应，这样一来其行为就类似与调用定义过的方法一样。

### 示例代码
```ruby
class Roulette
  def method_missing(name, *args)
    person = name.to_s.capitalize
    super unless %w[Bob Frank Bill Honda Eric].include? person
    number = 0
    3.times do
      number = rand(10) + 1
      puts "#{number}..."
    end
    "#{person} got a #{number}"
  end
end

number_of = Roulette.new
puts number_of.bob
puts number_of.kitty

```

### 动态代理

对一些封装过的对象，通过method_missing方法收集调用，并把这些调用转发到被封装的对象，这一过程称为动态代理,其中method_missing体现了动态，转发体现了代理

### const_missing方法

与method_missing类似，还有关于常量的const_missing方法，当引用一个不存在的常量时，Ruby会把这个常量名作为一个符号传递给const_missing方法。

### 白板类(blank slates)

拥有极少方法的类称为白板类，通过继承BasicObject类，可以迅速的得到一个白板类。除了这种方法以外，还可以通过删除方法来将一个普通类变为白板类。

#### 删除方法

删除某个方法有两种方式:

+ Module#undef_method
+ Module#remove_method

二者的区别是Module#undef_method会删除所有(包括继承而来的)方法。而Module#remove_method只删除接受者自己的方法，而保留继承来的方法。

### 动态方法与Method_missing的使用原则

当可以使用动态方法时候，尽量使用动态方法。除非必须使用method_missing方法(方法特别多的情况)，否则尽量少使用它。

### 代码块

Ruby中的代码块这个Topic，之前的文章有介绍过，需要的同学可以翻一翻历史消息，这里做个简短的回顾与总结

代码块的定义方式有{}花括号与do...end关键字定义两种，单行用花括号，多行用do...end

代码块只有在方法调用的时候才可以定义，块会被直接传递给这个方法，判断某个方法调用中是否包含代码块，可以通过Kernel#block_given?

代码块不仅可以有自己的参数，也会有返回值，往往没有代码块中的最后一行执行结果会被作为返回值返回

代码块之所以可以执行，是因为其不仅包含代码，同时也涵盖一组相应绑定，即执行环境，也可以称之为上下文环境。代码块可以携带这个上下文环境，到任何一个代码块可以达到的地方。也就可以说，一个代码块是一个闭包，当定义一个代码块时，它会捕获当前环境中的绑定，并带它们四处流动。

###  作用域

Ruby中不具备嵌套作用域(即在内部作用域，可以看到外部作用域的)的特点，它的作用域是截然分开的，一旦进入一个新的作用域，原先的绑定会被替换为一组新的绑定。

程序会在三个地方关闭前一个作用域，同时打开一个新的作用域，它们是:

+ 类定义class
+ 模块定义 module
+ 方法定义 def

上面三个关键字，每个关键字对应一个作用域门(进入)，相应的end则对应离开这道门。

#### 扁平化作用域

从一个作用域进入另一个作用域的时候，局部变量会立即失效，为了让局部变量持续有效，可以通过规避关键字的方式，使用方法调用来代替作用域门，让一个作用域看到另一个作用域里的变量，从而达到目的。具体做法是，通过Class.new替代class，Module#define_method代替def,Module.new代替module。这种做法称为扁平作用域，表示两个作用域挤压到一起。

#####示例代码(Wrong)
```ruby
my_var = “Success”
class MyClass
    puts my_var  #这里无法正确打印”Success”
    def my_method
        puts my_var  #这里无法正确打印”Success”
    end
end
```

##### 示例代码(Right)
```ruby
my_var = “Success”
MyClass = Class.new do
    puts “#{my_var} in  the class definition”
    define_method :my_method do
        “#{my_var} in the method”
    end
end
```

#### 共享作用域

将一组方法定义到，某个变量的扁平作用域中，可以保证变量仅被有限的几个方法所共享。这种方式称为共享作用域

### instance_eval方法

这个BasicObject#instance_eval有点类似JS中的bind方法，不同的时，bind是将this传入到对象中，而instance_eval则是将代码块(上下文探针Context Probe)传入到指定的对象中，一个是传对象，一个是传执行体。通过这种方式就可以在instance_eval中的代码块里访问到调用者对象中的变量。

#### 示例代码
```ruby
class MyClass
    def initialize
        @v = 1
    end
end
obj = MyClass.new

obj.instance_eval do
    self    #=> #<MyClass:0x33333 @v=1>
    @v      #=> 1  
end

v = 2
obj.instance_eval { @v = v }
obj.instance_eval { @v }   # => 2
```

此外，instance_eval方法还有一个双胞胎兄弟：instance_exec方法。相比前者后者更加灵活，允许对代码块传入参数。

#### 示例代码
```ruby
class C
    def initialize
        @x = 1
    end
end
class D
    def twisted_method
        @y = 2
        #C.new.instance_eval { “@x: #{@x}, @y>: #{y}” }
        C.new.instance_exec(@y) { |y| “@x: #{@x}, @y: #{y}” }
    end
end
#D.new.twisted_method   # => “@x: 1, @y: ”
D.new.twisted_method   # => “@x: 1, @y: 2”
```

因为调用instance_eval后，将调用者作为了当前的self，所以作用域更换到了class C中，之前的作用域就不生效了。这时如果还想访问到之前@y变量，就需要通过参数打包上@y一起随instance_eval转义，但因为instance_eval不能携带参数，所以使用其同胞兄弟instance_exec方法。

### 洁净室

一个只为在其中执行块的对象，称为洁净室，其只是一个用来执行块的环境。可以使用BasicObject作为洁净室是个不错的选择，因为它是白板类，内部的变量和方法都比较有限，不会引起冲突。

### Proc对象

Proc是由块转换来的对象。创建一个Proc共有四种方法,分别是:

#### 示例代码

```ruby
# 法一
inc = Proc.new { | x | x + 1}
inc.call(2)  #=> 3

# 法二
inc = lambda {| x | x + 1 }
inc.call(2)  #=> 3

# 法三
inc = ->(x) { x + 1}
inc.call(2) #=> 3

# 法四
inc = proc {|x| x + 1 }
inc.call(2) #=> 3
```

除了上面的四种之外，还有一种通过&操作符的方式，将代码块与Proc对象进行转换。如果需要将某个代码块作为参数传递给方法，需要通过为这个参数添加&符号，并且其位置必须是在参数的最后一个

&符号的含义是： 这是一个Proc对象，我想把它当成代码块来使用。去掉&符号，将能再次得到一个Proc对象。

#### 示例代码
```ruby
def my_method(&the_proc)
    the_proc
end

p = my_method {|name| “Hello, #{name} !”}
p.class   #=> Proc
p.call(“Bill”)   #=> “Hello,Bill”


def my_method(greeting)
    “#{greeting}, #{yield}!”
end

my_proc = proc { “Bill” }
my_method(“Hello”, &my_proc)
```

### Proc与Lambda对比

使用Lambda方法创建的Proc与其它方式创建的Proc是有一些差别的，用lambda方法创建的Proc称为lambda,而用其他方式创建的则称为proc。通过Proc#lambda?可以检测Proc是不是lambda。

二者之间主要的差异有以下两点:

+ Proc、Lambda的return含义不同; lambda中的return表示的仅仅是从lambda中返回。而proc中，return的行为则不同，其并不是从proc中返回，而是从定义proc的作用域中返回。即相当与在你的业务代码处返回。

#### 示例代码
```ruby
def double(callable_object)
    p = Proc.new { return 10 }
    result = p.call   
    return result * 2 # 不可达的代码
end

double #=> 10
```

+ Proc、Lambda的return参数检查方式不同；Proc的参数检查要比Lambda参数检查要更宽松一些，如果传入Proc中的参数数量不匹配其不会发生报错，会自行进行一定的调整到期望参数的样子，但是对于lambda则不同，如果出现参数不匹配的情况，其往往会报ArgumentError异常，中断程序。
#### 示例代码
```ruby
p = Proc.new { |a, b|  [a,b] }
p.call(1,2,3) #=> [1, 2]
p.call(1) #=> [1, nil]
```

### Proc与Lambda之间的选择

lambda更直观，更像是一个方法，参数要求更加严格，return也更像是方法定义中的return，往往Ruby程序员会将lambda作为第一选择。（PS: 这部分还真需要考证一下！）

### 可调用对象

+ 代码块(不是真正的对象，但是它们是”可调用的”):在定义它们的作用域中执行
+ proc: Proc类的对象与代码块一样，也在定义自身的作用域中执行
+ lambda: 同样是Proc类的对象，但跟普通的proc有细微差别。与代码块与proc一样都是闭包，也在定义自身的作用域中执行
+ 方法: 绑定于一个对象，在 所绑定对象的作用域中执行。他们也可以与这个作用域解除绑定，然后再重新绑定到另一个对象的作用域中
前三个在之前的笔记中都有详细介绍，今天来详细看下最后一个Method对象部分

#### Method 对象

通过Kernel#method方法，可以获得一个用Method对象表示的方法，在之后可以用Method#call方法对其进行调用。同样也可以用Kernel#singleton_method方法把单件方法名转换为Method对象

##### 示例代码
```ruby
class MyClass
    def initialize(value)
        @x = value
    end
    def my_method
        @x
    end
end

obj = MyClass.new(1)
m = obj.method :my_method
m.call   #=> 1
```

#### 自由方法

与普通方法类似，不同的是它会从最初定义它的类或模块中脱离出来(即脱离之前的作用域)，通过Method#unbind方法，可以将一个方法变为自由方法。同时，你也可以通过调用Module#instance_method获得一个自由方法.

###### 示例代码
```ruby
module MyClass
    def my_method
        42
    end
end

unbound = MyModule.instance_method(:my_method)
unbound.class  #=> UnboundMethod
```

上面代码中的UnboundMethod对象，不能够直接用来调用，不过可以将一个UnboundMethod方法绑定到一个对象上，使其再次成为一个Method对象，然后再被调用执行。

上面的绑定过程可以通过UnboundMethod#bind方法把UnboundMethod对象绑定到一个对象上，从某个类中分离出来的UnboundMethod对象只能绑定在该类及其子类对象上，不过从模块中分离出来的UnboundMethod对象在Ruby2.0后就不在有此限制了。

此外还可以将UnboundMethod对象传递给Module#define_method方法，从而实现绑定。

```ruby
String.send :define_method, :another_method, unbound
“abc”.anther_method  #=> 42

### 类的返回值

像方法一样，类定义也会返回最后一条语句的值:
```ruby
result = class MyClass
    self
end
result #=> MyClass
```

### 当前类

与当前对象self一样，同时还存在**当前类（或模块）**

Ruby中并没有类似当前对象self一样的明确引用，不过在追踪当前类的时候，可以遵循下面几条:

+ 在程序顶层，当前类是Object，这是main对象所属的类。
+ 在一个方法中，当前类就是当前对象的类。(在一个方法中用def关键字定义另一个方法，新定义的方法会定义在self所属的类中。因为Ruby解释器总是追踪当前类(或模块)的引用，所以使用def定义的方法都成为当前类的实例方法了)

#### 示例代码
```ruby
class C
    def m1
        def m2; end
    end
end

class D < C; end

obj = D.new
obj.m1

C.instance_methods(false)   #=> [:m1, :m2]
```

+ 当用class关键字打开一个类时(或module关键字打开模块)，那个类称为当前类。

### class_eval方法

Module#class_eval方法(别名: module_eval)，会在一个已存在的类的上下文中执行一个块。使用该方法可以在不需要class关键字的前提下，打开类。

Module#class_eval与Object#instance_eval方法相比，后者instance_eval方法只能修改self，而class_eval方法可以同时修改self与当前类。

此外class_eval的另一个优势就是可以利用扁平作用域，规避class关键字的作用域门问题。

#### instance_eval 与 class_eval的选择

通常使用instance_eval方法打开非类的对象，而用class_eval方法打开类定义，然后用def定义方法。

### 单件方法

Ruby允许给单个对象增加方法,这种只针对单个对象生效的方法，称为单件方法

##### 示例代码
```ruby
str = “just a regular string”

def str.title?
    self.upcase == self
end

str.title?  # => false
str.methods.grep(/title?/) # => [:title?]
str.singleton_methods   #=> [:title?]

str.class # => String
String.title?  #=>  NoMethodError
```

另外，除了上面使用的定义方法，还可以通过Object#define_singleton_method方法来定义单件方法

#### 单件方法与类方法

前面的笔记中有说道在Ruby中类也是对象，而类名只是常量，所以，在类上调用方法其实跟在对象上调用方法一样:

类方法的实质是: 它是一个类的单件方法，实际上，如果比较单件方法的定义和类方法的定义，会发现其实二者是一样的

```ruby
def obj.a_singleton_method; end
def MyClass.another_class_method; end
```

二者均使用了def关键词做定义

```ruby
def object.method
    #方法主体
end
```

上面的object可以是*对象的引用、常量类名或者self。

### 类宏attr_accessor

Ruby对象没有属性，如果希望得到一些像属性的东西，需要分别定义一个读方法和写方法（也就是java、objc中的set和get方法)，最直接的可以这样:

##### 示例代码
```ruby
class MyClass
    def my_attribute=(value)
        @my_attribute =value    
    end
    def my_attribute
        @my_attribute
    end
end

obj = MyClass.new
obj.my_attribute = ‘x’
obj.my_attribute    #=> ‘x’
```

但是上面这种写法，如果属性众多的话就会存在Repeat Yourself的地方，这时就可以用到下面三个类宏:

+ Module#attr_reader 生成一个读方法
+ Module#attr_writer 生成一个写方法
+ Module#attr_accessor 同时生成读方法和写方法

##### 示例代码
```ruby
class MyClass
    attr_accessor :my_attribue
end
```

这样是不是就简洁多了呢? 当然，使用方法(读与写)跟上面的实现是一致的。

### 单件类

我们知道Ruby中对象的方法的查找顺序是: **先向右，再向上**，其含义就是先向右找到对象的类，先在类的实例方法中尝试查找，如果没有找到，再继续顺着**祖先链**找。

前面一篇中有介绍过**单件方法**，单件方法是指那些只针对某个对象有效的方法，那么如果为一个对象定义了单件方法，那么这个单件方法的查找顺序又应该是怎样的？

```ruby
class MyClass
    def my_method; end
end

obj = MyClass.new

def obj.my_singleton_method; end
```

首先,**单件方法**不会在obj中，因为obj不是一个类，其次它也不在MyClass中，那样的话所有的MyClass都应该能共享调用这个方法，也就构不成单件类了。同理，单件方法也不能在**祖先链**的某个位置(类似superclass: Object)中。正确的位置是在**单件类**中，这个类其实就是我们在irb中向对象询问它的类时(obj.class)得到的那个类，不同的是这类与普通的类还是有稍稍不同的。也可以称其为**元类或本征类**。

### 打开单件类

Ruby提供了两种方法获取单件类的引用，一种是通过传统的关键词class配合特殊的语法

#### 法一
```ruby
class << an_object
    # 自己的代码
end

obj = Object.new
singleton_class = class << obj
    self
end
singleton_class.class  # => Class
```

另一个方法是，通过Object#singleton_class方法来获得单件类的引用:

#### 法二
```ruby
“abc”.singleton_class   # => #<Class: #<String:0xxxxxx>>
```

### 单件类的特性

+ 每个单件类只有一个实例（被称为单件类的原因），而且不能被继承
+ 单件类是一个对象的单件方法的存活所在

### 引入单件类后的方法查找

基于上面对单件类的基本认识，引入单件类后，Ruby的方法查找方式就不应该是先从其类(普通类)开始，而是应该先从对象的单件类中开始查找，如果在单件类中没有找到想要的方法，它才会开始沿着类(普通类)开始，再到祖先链上去找。这样从单件类之后开始，一切又回到了我们在没有引入单件类时候的次序。

通过下面这个代码可以自行验证一下
```ruby
class C
  def a_method
    "C#a_method()"
  end
end

class D < C; end

obj = D.new
```

### 打开单件类定义单件方法

```ruby
class << obj
  def a_singleton_method
    obj #a_singleton_method()
  end
end

obj.singleton_class.superclass  #=> D
```

### Ruby对象模型的7条规则

1. 只有一个对象 —— 要么是普通对象，要么是模块
2. 只有一个模块 —— 可以是一个普通模块、一个类或者一个单件类
3. 只有一个方法，它存在于一个模块中 —— 通常是一个类中
4. 每个对象（包括类）都有自己的“真正的类”—— 要么是一个普通类，要么是一个单件类
5. 除了BasicObject类没有超类外，每个类有且只有一个祖先(superclass) —— 要么是一个类，要么是一个模块。这意味着任何类只有一条向上的，直到BasicObject的祖先链
6. 一个**对象**的单件类的超类是这个对象的类；一个**类**的单件类的超类是这个类的超类的单件类。(这条建议结合后面的附图来看！)
7. 调用一个方法时，Ruby先向右迈出一步进入接收者真正的类(单件类)，然后向上进入祖先链。

### 类方法

类方法其实质是生活在该类的单件类中的单件方法。其定义方法有三种，分别是:

```ruby
# 法一
def MyClass.a_class_method; end


# 法二
class MyClass
  def self.anther_class_method; end
end


# 法三*
class MyClass
  class << self
    def yet_another_class_method; end
  end
end
```

其中第三种方法道出了，类方法的实质，特别记忆一下！

### 模块与单件类

当一个类包含一个模块时，他获得的是该模块的实例方法，而不是类方法。类方法存在与模块的单件类中，没有被类获得。

### 类扩展

类扩展通过向类的单件类中添加模块来定义类方法。

```ruby
module MyModule
  def my_method; 'hello'; end
end

class MyClass
  class < self
    include MyModule
  end
end

MyClass.my_method
```

上面代码展示了具体**类扩展**的实现方式，将一个MyModule模块引入到MyClass类的单件类中，因为my_method方法是MyClass的单件类的一个实例方法，这样，my_method方法也是MyClass的一个类方法。

### 对象扩展

**类方法**是**单件方法**的特例，因此可以把类扩展这种技巧应用到**任意对象**上，这种技巧即为**对象扩展**

```ruby
# 法一: 打开单件类来扩展
module MyModule
  def my_method; 'hello'; end
end

obj = Object.new
class << obj
  include MyModule
end

obj.my_method   # => “hello”
obj.singleton_methods   # => [:my_method]
# 法二：Object#extend方法
module MyModule
  def my_method; 'hello'; end
end

obj = Object.new
#对象扩展
obj.extend MyModule
obj.my_method   # => “hello”
#类扩展
class MyClass
  extend MyModule
end

MyClass.my_method  # => “hello”
```

Object#extend是在接受者的单件类中包含模块的快键方式。

### 方法包装器（Method Wrapper）

方法包装器一般适合于，处理一些你不能直接触碰到的方法，或者需要给某个方法进行一些预处理工作，这个概念有点类似与Python中的装饰器

#### 方法别名

Ruby中使用Module#alias_method方法和alias关键字为方法取别名。

##### 示例代码
```ruby
class MyClass
  def my_method
    "my_method()"
  end
  alias_method :m, :my_method
end

obj = MyClass.new
obj.my_method    #=> “my_method()”
obj.m   #=> “my_method()”
```

需要注意的是，在顶级作用域中（main）中只能使用alias关键字来命名别名，因为在那里调用不到Module#alias_method方法

此外这里，还需要指出一点关于**方法重定义**的理解误区，一般我们认为方法一旦被重定义了，就找不回原来的方法了，其实在Ruby中的重定义只是，对方法名与方法实体绑定的一种重新绑定，如果被重新定义的方法体还有一个其它别名，那么一样还是可以访问到老方法的。

有了上面这个理解，我们来看**环绕别名**这个概念。

#### 环绕别名

经过下面三个步骤可以实现**环绕别名**

1. 给方法定义一个别名
2. 重定义这个方法
3. 在新的方法中调用老的方法

上面三步后，即可以保留老方法(第一步)，同时又为原方法增加了新功能。但其本质上没有改变调用之间的函数名称依赖，是一种名副其实的“新瓶装老酒”的取巧办法。

不过**环绕别名**也有缺点，就是它污染了类的命名空间，添加了额外的方法名。不过这一点其实可以通过Ruby中的private关键字把旧方法限定为私有方法。(Ruby中的public和private实际上针对的是方法名，而非方法本身，言外之意为private方法取个别名，也可以把私有方法暴露出来)

最后要注意的是不要尝试load两次**环绕别名**的实现代码，否则得到的执行结果就不是你预期中的了，不信你可以试着连续走两遍上面的3个步骤。

#### 细化

前面提到的细化，也可以作为一种方法包装器，在细化的方法中调用super方法，可以调用会细化之前的方法。此外细化相比起**环绕别名**，细化的作用范围是文件末尾，而环绕别名则是作用在全局。

```ruby  
module StringRefinement
  refine String do
    def length
      super > 5 ? 'long' : 'short'
    end
  end
end

using StringRefinement

puts "War and Peace".length  #=> “long”
```

#### 下包含包装器 (Module#prepend)

在介绍**祖先链**的部分有涉及过include与prepend两个方法，前者是将模块插入到当前类的上方，后者是插入到下方，而下方的位置，正好是方法查找时优先查找的位置，利用这一优势，可以覆写当前类的同名方法，同时通过super关键字还可以调用到该类中的原始方法

```ruby
module ExplicitString
  def length
    super > 5 ? 'long' : 'short'
  end
end

String.class_eval do
  prepend ExplicitString
end

puts "War and Peace".length  #=> “long”
```

### Kernal#eval方法

Kernal#eval方法与之前的BasicObject#instance_eval和Module#class_eval一样，都属于*eval家族，都可以赋予程序在运行中进行动态变化的能力。与后两者想比Kernal#eval更加直接，不需要代码块、直接就可以执行字符串代码(String of Code)。

PS:BasicObject#instance_eval也是可以执行字符串代码的。

#### 示例代码
```ruby
array = [10, 20]
element = 30
eval(“array << element”)  #=> [10, 20, 30]
array.instance_eval "self << element"  #=> [10, 20, 30]
```

### Here文档(Here documents)

以<<打头，后面跟一个“结束序列标识”，之后就可以是正式的文档内容了，可以任意换行，直到遇到了独立出现的”结束序号标识”。
```ruby
puts <<GROCERY_LIST
Grocery list
------------
1. Salad mix.
2. Strawberries.*
3. Cereal.
4. Milk.*

* Organic
GROCERY_LIST
```

上面代码的输出格式如下:
```ruby
Grocery list
------------
1. Salad mix.
2. Strawberries.*
3. Cereal.
4. Milk.*

* Organic
=> nil
```

### 绑定对象

Binding是一个用对象标识的完整作用域(上下文环境)。可以通过创建Binding对象来捕获并带走当前的作用域。之后，通过eval方法在这个Binding对象所携带的作用域内执行代码。

使用Kernel#binding方法可以用来创建Binding对象

#### 示例代码
```ruby
class MyClass
  def my_method
    @x = 1
    binding
  end
end

b = MyClass.new.my_method
eval “@x”, b #=> 1
```

上面代码，在MyClass类中定义了一个my_method方法来返回一个当前的绑定。最后将这个返回的绑定，作为参数传递给eval方法。这样“@x” 就可以在返回的绑定作用域中执行了。

关于绑定还有另外一个知识点，Ruby还提供了一个名为TOPLEVEL_BINDING的预定义常量，表示顶级作用域Binding对象。该常量可以在程序的任何位置访问到。言外之意，你可以在程序的任何位置，通过Kernal#eval方法在顶级作用域中执行代码。

#### 示例代码
```ruby
class AnotherClass
  def my_method
    eval "self", TOPLEVEL_BINDING
  end
end

AnotherClass.new.my_method  #=> main
```

#### 代码注入

Kernal#eval执行这样可以灵活执行字符串代码的特性，给编程带来了灵活性之外，也带来了潜在的风险，如果字符串代码来源于不可信的用户输入，如果不做安全检查，保不齐什么时候就会是一段破坏性的恶意代码。

面对这样的风险，可以选择规避eval的使用，换用其它相对安全的方式代替,例如**动态方法**和**动态派发**。此外，还可以通过在Ruby中通过修改$SAFE全局变量值，来控制程序的安全性级别，具体就是在你要执行**可信的**字符串代码前，将安全级别降低，可以使用Object#untaint方法，执行完之后在切换安全级别。这有点像操作系统中使用临界资源的步骤(请求锁，释放锁)

通过Object#tainted?方法可以判断一个对象是不是被污染了（是否来自一个不可信的输入源）。

### 钩子方法

钩子方法有些类似事件驱动装置，可以在特定的事件发生后执行特定的回调函数，这个回调函数就是**钩子方法**(更形象的描述: 钩子方法可以像钩子一样，勾住一个特定的事件。)，在Rails中before\after函数就是最常见的钩子方法。

Class#inherited方法也是这样一个钩子方法，当一个类被继承时，Ruby会调用该方法。默认情况下，Class#inherited什么都不做，但是通过继承，我们可以拦截该事件，对感兴趣的继承事件作出回应。
```ruby
class String
  def self.inherited(subclass)
    puts "#{self} was inherited by #{subclass}"
  end
end
class MyString < String; end
#输出
String was inherited by MyString
```

通过使用钩子方法，可以让我们在Ruby的类或模块的生命周期中进行干预，可以极大的提高编程的灵活性。

这些生命周期相关的钩子方法还有下面这些:

#### 类与模块相关

+ Class#inherited
+ Module#include
+ Module#prepended
+ Module#extend_object
+ Module#method_added
+ Module#method_removed
+ Module#method_undefined

#### 单件类相关

+ BasicObject#singleton_method_added
+ BasicObject#singleton_method_removed
+ BasicObject#singleton_method_undefined

##### 示例代码
```ruby
module M1
  def self.included(othermod)
    puts "M1 was included into #{othermod}"
  end
end

module M2
  def self.prepended(othermod)
    puts "M2 was prepended to #{othermod}"
  end
end

class C
  include M1
  include M2
end

# 输出
M1 was included into C
M2 was prepended to C

module M
  def self.method_added(method)
    puts "New method: M##{method}"
  end

  def my_method; end
end

# 输出
New method: M#my_method
```

除了上面列出来的一些方法外，也可以通过重写父类的某个方法，进行一些过滤操作后，再通过调用super方法完成原函数的功能，从而实现类似钩子方法的功效，如出一辙，**环绕别名**也可以作为一种钩子方法的替代实现。

前面已经对元编程的理论知识走了一遍，后面的内容侧重于实际操作和源码阅读，今天的Topic就是由书中第六章中的例子而来。

书中以迭代的形式，引导读者一步步实现一个名为attr_checked的类宏，正所谓上帝造人也需要7天，创造和生产这件事儿，过程和迭代都是少不了的，本文完全遵照guide而来，美中不足的地方是没有添加，测试用例，不过每个步骤都给出了一些简单的实例演示代码。

### 任务描述
写一个操作方法类似`attr_accessor`的`attr_checked`的类宏，该类宏用来对属性值做检验，使用方法如下:

```ruby
class Person
  include CheckedAttributes

  attr_checked :age do |v|
    v >= 18
  end
end

me = Person.new
me.age = 39  #ok
me.age = 12  #抛出异常
```

### 实施计划
1. 使用eval方法编写一个名为`add_checked_attribute`的内核方法，为指定类添加经过简单校验的属性
2. 重构add_checked_attribute方法，去掉eval方法，改用其它手段实现
3. 添加代码块校验功能
4. 修改add_checked_attribute为要求的attr_checked,并使其对所有类都可用
5. 通过引入模块的方式，只对引入该功能模块的类添加attr_checked方法


#### Step 1
```ruby
def add_checked_attribute(klass, attribute)
  eval "
    class #{klass}
      def #{attribute}=(value)
        raise 'Invalid attribute' unless value
        @#{attribute} = value
      end
      def #{attribute}()
        @#{attribute}
      end
    end
  "
end

add_checked_attribute(String, :my_attr)
t = "hello,kitty"

t.my_attr = 100
puts t.my_attr

t.my_attr = false
puts t.my_attr
```

这一步使用`eval`方法，用`class`和`def`关键词分别打开类，且定义了指定的属性的get和set方法，其中的set方法会简单的判断值是否为空(nil 或 false)，如果是则抛出`Invalid attribute`异常。

#### Setp 2
```ruby
def add_checked_attribute(klass, attribute)
  klass.class_eval do
    define_method "#{attribute}=" do |value|
      raise "Invaild attribute" unless value
      instance_variable_set("@#{attribute}", value)
    end

    define_method attribute do
      instance_variable_get "@#{attribute}"
    end

  end
end

```

这一步更换掉了`eval`方法,同时也分别用`class_eval`和`define_method`方法替换了之前的`class`与`def`关键字，实例变量的设置和获取分别改用了`instance_variable_set`和`instance_variable_get`方法,使用上与第一步没有任何区别，只是一些内部实现的差异。

### Step 3
```ruby
def add_checked_attribute(klass, attribute, &validation)
  klass.class_eval do
    define_method "#{attribute}=" do |value|
      raise "Invaild attribute" unless validation.call(value)
      instance_variable_set("@#{attribute}", value)
    end

    define_method attribute do
      instance_variable_get "@#{attribute}"
    end

  end
end

add_checked_attribute(String, :my_attr){|v| v >= 180 }
t = "hello,kitty"

t.my_attr = 100  #Invaild attribute (RuntimeError)
puts t.my_attr

t.my_attr = 200
puts t.my_attr  #200
```
没有什么奇特的，只是加了通过**代码块**验证，增加了校验的灵活性，不再仅仅局限于nil和false之间了。


### Step 4
```ruby
class Class
  def attr_checked(attribute, &validation)
      define_method "#{attribute}=" do |value|
        raise "Invaild attribute" unless validation.call(value)
        instance_variable_set("@#{attribute}", value)
      end

      define_method attribute do
        instance_variable_get "@#{attribute}"
      end
  end
end

String.add_checked(:my_attr){|v| v >= 180 }
t = "hello,kitty"

t.my_attr = 100  #Invaild attribute (RuntimeError)
puts t.my_attr

t.my_attr = 200
puts t.my_attr  #200

```

这里我们把之前顶级作用域中方法名放到了`Class`中，由于所有对象都是`Class`的实例, 所以这里定义的实例方法，也能被Ruby中的其它所有类访问到，同时在class定义中,self就是当前类，所以也就省去了调用类这个参数和`class_eval`方法，并且我们把方法的名字也改成了`attr_checked`。

### Step 5
```ruby
module CheckedAttributes
  def self.included(base)
    base.extend ClassMethods
  end
end

module ClassMethods
  def attr_checked(attribute, &validation)
      define_method "#{attribute}=" do |value|
        raise "Invaild attribute" unless validation.call(value)
        instance_variable_set("@#{attribute}", value)
      end

      define_method attribute do
        instance_variable_get "@#{attribute}"
      end
  end
end

class Person
  include CheckedAttributes

  attr_checked :age do |v|
    v >= 18
  end
end
```

最后一步通过**钩子方法**，在`CheckedAttributes`模块被引入后,对当前类通过被引入模块进行扩展，
从而使当前类支持引入后的方法调用，即这里的get与set方法组。

到此，我们已经得到了一个名为`attr_checked`，类似`attr_accessor`的类宏，通过它你可以对属性进行你想要的校验。

前面介绍了Ruby元编程中的各种黑魔法，今天这个Topic被作者归纳为**惯用法**，看了以后帮助巨大！这些用法还真是经常出现在各种Ruby大神的代码里，Ruby-China的源码中就不少,不啰嗦进入正题！

### 拟态方法

拟态方法其实就是我们常见类似puts、private以及public这些方法，它们将自己伪装成类似关键字的样子，但其实它们都是Ruby中的方法。在Rails中也有大量的这样用法，比如link_to这样的方法，常常让人搞不清楚这个方法到底可以接收多少个参数？其实它的定义是这样的:

```ruby
link_to(name = nil, options = nil, html_options = nil, &block)
```

之所以感觉它的参数多的原因是，其省略了hash参数的花括号，**Ruby 中如果最後一個參數是 Hash 的話，它的大括號是可以省略的**，所以你才见到link_to参数总是数不完。

### 空指针保护

相信有读过大神代码的同学，一定遇到过这样的赋值语句吧a ||= [], 这里的重点可以放到||=上，特别是在Rails中通过参数对变量赋值什么的。其实它的含义可以用这个表达式表示a || (a = []) ,这里利用了布尔运算的短路求值。等价于下面这段代码：

```ruby
if defined?(a) && a
  # 如果a已经被定义了，且既不为nil也非false，返回a
  a
else
  # 如果a未定义，或未nil，或为false，返回空数组
  a = []
end
```

上面代码，明显表示出，a ||= []这样的表达式，除了能确保变量初始化以外，还能确保其值不为nil和false。所以，不应该在变量值可能是nil或false的情况下使用**空指针保护**

**空指针保护**常用于初始化变量,可以取代initialize方法。如下面代码:

```ruby
class C
  def initialize
    @a = []
  end
  def elements
    @a
  end
end
```

上面代码通过，initialize方法对实例变量@a进行了初始化，确保在调用C#elements方法时，@a是已经初始化的。同样的结果，使用**惰性实例变量**技巧，可以让代码更精简

```ruby
class C
  def elements
    @a || = []
  end
end
```

只有在C#elements被调用的时候，才初始化实例变量@a。

### Self Yield

给一个方法传入代码块时，可以通过yield占位对块进行回调。除了直接调用块外，还可以通过yeild给代码块传递参数，这个self yield的惯用法，其实就是通过yield self，把自身(当前对象)传递给代码块。

#### 使用tap调试方法调用链
```ruby
['a','b','c'].push('d').shift.tap{|x| puts x}.upcase.next
#输出
a
```

这里的tap方法就是利用了self yield技术，将自己传给代码块，从而让我们能打印出当前的状态值，辅助调试方法调用链，避免“火车失事”问题。

Kernel#tap的定义和文档描述如下:
```ruby
tap{|x|...} → obj
```

Yields self to the block, and then returns self.** The primary purpose of this method is to “tap into” a method chain, in order to perform operations on intermediate results within the chain.**

不过自己实现一个tap方法也不难:
```ruby
class Object
  def tap
    yield self
    self
  end
end
```

### Symbol#to_proc方法

Symbol#to_proc方法的目的是位了用更简单的方式来替代一次调用代码块。所谓的一次调用代码块是指那些只有一个参数，且对这个参数只调用一个方法。如下:
```ruby
names = ['bob', 'bill', 'heather']
names.map{ | name | name.capitalize } #=> [“Bob”, “Bill”, “Heather”]
```

那么Symbol#to_proc方法又是怎么做到精简的呢？继续往下看
```ruby
names = ['bob', 'bill', 'heather']
names.map(&:capitalize) #=> [“Bob”, “Bill”, “Heather”]
```

上面出现了&符号，&符号的含义是： 这是一个Proc对象，我想把它当成代码块来使用。去掉&符号，将能再次得到一个Proc对象。&符号可以用作任何对象，它会调用该对象的to_proc方法来把这个对象转换为一个Proc，之后map方法会将names数组中的每个值作为这个Proc对象的参数进行调用。
```ruby
class Symbol
    def to_proc
        Proc.new { |x| x.send(self) }
    end
end
```

用到了动态派发，to_proc方法会返回一个Proc对象，该对象会在传入的参数上执行:method_name（符号方法，因为动态调用的是self，符号对象本身）。

此外，Ruby中的对象还支持多于一个参数的块，类似inject方法(ps: 这里还不是很熟),见下面代码。
```ruby
[1,2,5].inject(0) { | memo, obj | memo + obj } #=> 8
[1,2,5].inject(0, &:+) #=> 8
```

