# smali

## 寄存器

Dalvik 指令集完全基于寄存器，也就是说，没有栈。

所有寄存器都是 32 位，无类型的。也就是说，虽然编译器会为每个局部变量分配一个寄存器，但是理论上一个寄存器中可以存放一个`int`，之后存放一个`String`（的引用），之后再存放一个别的东西。

如果要处理 64 位的值，需要连续的两个寄存器，但是代码中仍然只写一个寄存器。这种情况下，你在代码中看到的`vx`实际上是指`vx`和`vx + 1`。

寄存器有两种命名方法。`v`命名法简单直接。假设一共分配了 10 个寄存器，那么我们可以用`v0`到`v9`来命名它们。

局部变量可以使用的寄存器：v0,v1  
参数可以使用的寄存器：v2,v3,v4

除此之外，还可以用`p`命名法来命名参数所用的寄存器，参数会占用后面的几个寄存器。假如上面那个方法是共有两个参数的静态方法，那么，我们就可以使用`p0`和`p1`取代`v8`和`v9`。如果是实例方法，那么可以用`p0 ~ p2`取代`v7 ~ v9`，其中`p0`是`this`引用。

局部变量可以使用的寄存器：v0,v1  
参数可以使用的寄存器：p0,p1,p2

但在实际的代码中，一般不会声明所有寄存器的数量，而是直接声明局部变量所用的寄存器（后面会看到）。也就是说局部变量和参数的寄存器是分开声明的。我们无需关心`vx`是不是`py`，只需知道所有寄存器的数量是局部变量与参数数量的和。

## 数据类型

Dalvik 拥有独特的数据类型表示方法，并且和 Java 类型一一对应：

| Java 类型 | Dalvik 表示 |
| --- | --- |
| `boolean` | Z |
| `byte` | B |
| `short` | S |
| `char` | C |
| `int` | I |
| `long` | J |
| `float` | F |
| `double` | D |
| `void` | V |
| 对象类型 | L |
| 数组类型 | \[ |

其中对象类型由`L<包名>/<类名>;`（完全限定名称）表示，要注意末尾有个分号，比如`String`表示为`Ljava/lang/String;`。

数组类型是`[`加上元素类型，比如`int[]`表示为`[I`。左方括号的个数也就是数组的维数，比如`int[][]`表示为`[[I`。

## 类定义

一个 smali 文件中存放一个类，文件开头保存类的各种信息。类的定义是这样的。

```smali
.class <权限修饰符> <非权限修饰符> <完全限定名称>
.super <超类的完全限定名称>
.source <源文件名>
```

比如这是某个`MainActivity`：

```smali
.class public Lnet/flygon/myapplication/MainActivity;
.super Landroid/app/Activity;
.source "MainActivity.java"
```

我们可以看到该类是`public`的，完整名称是`Lnet.flygon.myapplication.MainActivity;`，类名以 `L` 开头 以 `;` 结尾。继承了`android.app.Activity`，在源码中是`MainActivity.java`。如果类是`abstract`或者`final`的，会在`public/private/protected`后面表示。

类可以实现接口，如果类实现了接口，那么这三条语句下面会出现`.implements <接口的完全限定名称>`。比如通常用于回调的匿名类中会出现`.implements Landroid/view/View$OnClickListener;`。

类还可以拥有注解，同样，这三条语句下方出现这样的代码：

```smali
.annotation <完全限定名称>
    键 = 值
    ...
.end annotation
```

这些语句下面就是类拥有的字段和方法。

## 字段定义

字段定义如下：

```smali
.field <权限修饰符> <非权限修饰符> <名称>:<类型>
```

其中非权限修饰符可以为`final`或者`abstract`。

比如我在`MainActivity`中定义一个按钮：

```smali
.field private button1:Landroid/widget/Button;
```

## 方法定义

方法定义如下：

```smali
.method <权限修饰符> <非权限修饰符> <名称>(<参数类型>)<返回值类型>
    ...
.end method
```

要注意如果有多个参数，参数之间是紧密挨着的，没有逗号也没有空格。如果某个方法的参数是`int, int, String`，那么应该表示为`IILjava/lang/String;`。

### `.locals`

方法里面可以包含很多很多东西，可以说是反编译的重点。首先，方法开头处可能会含有局部变量个数声明和参数声明。`.locals <个数>`可以用于变量个数声明，比如声明了`.locals 10`之后，我们就可以直接使用`v0`到`v9`的寄存器。

### `.param`

另外，参数虽然也占用寄存器，但是声明是不在一起的。`.param px,"<名称>"`用于声明参数。不是必需的。

### `.prologue`

之后`.prologue`的下面是方法中的代码。代码是接下来要讲的东西。

### `.line`

代码之间可能会出现`.line <行号>`，用来标识 Java 代码中对应的行，不过这个是非强制性的，修改之后对应不上也无所谓。

### `.local`

还可能出现局部变量声明，`.local vx, "<名称>":<类型>`。这个也是非强制性的，只是为了让你清楚哪些是具名变量，哪些是临时变量。临时变量没有这种声明，照样正常工作。甚至你把它改成不匹配的类型（`int`改成`Object`），也可以正常运行。

## 数据定义

| 指令 | 含义 |
| --- | --- |
| const/4 vx,lit4 | 将 4 位字面值`lit4`（扩展为 32 位）存入`vx` |
| const/16 vx,lit16 | 将 16 位字面值`lit16`（扩展为 32 位）存入`vx` |
| const vx, lit32 | 将 32 位字面值`lit32`存入`vx` |
| const-wide/16 vx, lit16 | 将 16 位字面值`lit16`（扩展为 64 位）存入`vx`及`vx + 1` |
| const-wide/32 vx, lit32 | 将 32 位字面值`lit32`（扩展为 64 位）存入`vx`及`vx + 1` |
| const-wide vx, lit64 | 将 64 位字面值`lit64`存入`vx`及`vx + 1` |
| const/high16 v0, lit16 | 将 16 位字面值`lit16`存入`vx`的高位 |
| const-wide/high16, lit16 | 将 16 位字面值`lit16`存入`vx`和`vx + 1`的高位 |
| const-string vx, string | 将指字符串常量（的引用）`string`存入`vx` |
| const-class vx, class | 将指向类对象（的引用）`class`存入`vx` |

这些指令会在我们给变量赋字面值的时候用到。下面我们来看看这些指令如何与 Java 代码对应，以下我定义了所有相关类型的变量。

```java
boolean z = true;
z = false;
byte b = 1;
short s = 2;
int i = 3;
long l = 4;
float f = 0.1f;
double d = 0.2;
String str = "test";
Class c = Object.class;
```

编译之后的代码可能是这样：

```smali
const/4 v10, 0x1
const/4 v10, 0x0
const/4 v0, 0x1
const/4 v8, 0x2
const/4 v5, 0x3
const-wide/16 v6, 0x4
const v4, 0x3dcccccd    # 0.1f
const-wide v2, 0x3fc999999999999aL    # 0.2
const-string v9, "test"
const-class v1, Ljava/lang/Object;
```

我们可以看到，`boolean`、`byte`、`short`、`int`都是使用`const`系列指令来加载的。我们在这里为其赋了比较小的值，所以它用了`const/4`。如果我们选择一个更大的值，编译器会采用`const/16`或者`const`指令。然后我们可以看到`const-wide/16`用于为`long`赋值，说明`const-wide`系列指令用于处理`long`。

接下来，`float`使用`const`指令处理，`double`使用`const-wide`指令处理。以`float`为例，它的`const`语句的字面值是`0x3dcccccd`，比较费解。实际上它是保持二进制数据不变，将其表示为`int`得到的。

我们可以用这段 c 代码来验证。

```c
int main() {
    int i = 0x3dcccccd;
    float f = *(float *)&i;
    printf("%f", f);
    return 0;
}
```

结果是`0.100000`，的确是我们当初赋值的 0.1。

最后，`const-string`用于加载字符串，`const-class`用于加载类对象。虽然文档中写着“字符串的 ID”，但实际的反编译代码中是字符串字面值，比较方便。对于类对象来说，代码中出现的是完全先定名称。

### 数据移动

数据移动指令就是大名鼎鼎的`move`：

| 指令 | 含义 |
| --- | --- |
| move vx,vy | `vx = vy` |
| move/from16 vx,vy | `vx = vy` |
| move/16 vx,vy | `vx = vy` |
| move-wide vx,vy | `vx, vx + 1 = vy, vy + 1` |
| move-wide/from16 vx,vy | `vx, vx + 1 = vy, vy + 1` |
| move-wide/16 vx,vy | `vx, vx + 1 = vy, vy + 1` |
| move-object vx,vy | `vx = vy` |
| move-object/from16 vx,vy | `vx = vy` |
| move-object/16 vx,vy | `vx = vy` |
| move-result vx | 将小于等于 32 位的基本类型（`int`等）的返回值赋给`vx` |
| move-result-wide vx | 将`long`和`double`类型的返回值赋给`vx` |
| move-result-object vx | 将对象类型的返回值（的引用）赋给`vx` |
| move-exception vx | 将异常对象（的引用）赋给`vx`，只能在`throw`之后使用 |

`move`系列指令以及`move-result`用于处理小于等于 32 位的基本类型。`move-wide`系列指令和`move-result-wide`用于处理`long`和`double`类型。`move-object`系列指令和`move-result-object`用于处理对象引用。

另外不同后缀（无、`/from16`、`/16`）只影响字节码的位数和寄存器的范围，不影响指令的逻辑。

## 数据运算

### 二元运算

二元运算指令格式为`<运算类型>-<数据类型> vx,vy,vz`。其中算术运算的`type`可以为`int`、`long`、`float`、`double`四种（`short`、`byte`按`int`处理），位运算的只支持`int`、`long`，下同。

| 指令 | 运算类型 | 含义 |
| --- | --- | --- |
| 算术运算 |  |  |
| add- vx, vy, vz | 加法 | `vx = vy + vz` |
| sub- vx, vy, vz | 减法 | `vx = vy - vz` |
| mul- vx, vy, vz | 乘法 | `vx = vy * vz` |
| div- vx, vy, vz | 除法 | `vx = vy / vz` |
| rem- vx, vy, vz | 取余 | `vx = vy % vz` |
| 位运算 |  |  |
| and- vx, vy, vz | 与 | `vx = vy & vz` |
| or- vx, vy, vz | 或 | `vx = vy \| vz` |
| xor- vx, vy, vz | 异或 | `vx = vy ^ vz` |
| shl- vx, vy, vz | 左移 | `vx = vy << vz` |
| shr- vx, vy, vz | 算术右移 | `vx = vy >> vz` |
| ushr- vx, vy, vz | 逻辑右移 | `vx = vy >>> vz` |

我们可以查看如下代码：

```java
int a = 5,
    b = 2,
    c = a + b,
    d = a - b,
    e = a * b,
    f = a / b,
    g = a % b,
    h = a & b,
    i = a | b,
    j = a ^ b,
    k = a << b,
    l = a >> b,
    m = a >>> b;
```

编译后的代码可能为：

```smali
const/4 v0, 0x5
const/4 v1, 0x2
add-int v2, v0, v1
sub-int v3, v0, v1
mul-int v4, v0, v1
div-int v5, v0, v1
rem-int v6, v0, v1
and-int v7, v0, v1
or-int v8, v0, v1
xor-int v9, v0, v1
shl-int v10, v0, v1
shr-int v11, v0, v1
ushr-int v12, v0, v1
```

这里有个特例，当操作数类型是`int`，并且第二个操作数是字面值的时候，有一组特化的指令：

| 指令 | 运算类型 | 含义 |
| --- | --- | --- |
| 算术运算 |  |  |
| add-int/ vx, vy, | 加法 | `vx = vy + <litn>` |
| sub-int/ vx, vy, | 减法 | `vx = vy - <litn>` |
| mul-int/ vx, vy, | 乘法 | `vx = vy * <litn>` |
| div-int/ vx, vy, | 除法 | `vx = vy / <litn>` |
| rem-int/ vx, vy, | 取余 | `vx = vy % <litn>` |
| 位运算 |  |  |
| and-int/ vx, vy, | 与 | `vx = vy & <litn>` |
| or-int/ vx, vy, | 或 | `vx = vy\| <litn>` |
| xor-int/ vx, vy, | 异或 | `vx = vy ^ <litn>` |
| shl-int/ vx, vy, | 左移 | `vx = vy << <litn>` |
| shr-int/ vx, vy, | 算术右移 | `vx = vy >> <litn>` |
| ushr-int/ vx, vy, | 逻辑右移 | `vx = vy >>> <litn>` |

其中`<litn>`可以为`lit8`或`lit16`，即 8 位或 16 位的整数字面值。比如`int a = 0; a += 2;`可能编译为`const/4 v0, 0`和`add-int/lit8 v0, v0, 0x2`。

### 二元运算赋值

二元运算赋值指令格式为`<运算类型>-<数据类型>/2 vx,vy,vz`。

| 指令 | 运算类型 | 含义 |
| --- | --- | --- |
| 算术运算 |  |  |
| add-/2addr vx, vy | 加法赋值 | `vx += vy` |
| sub-/2addr vx, vy | 减法赋值 | `vx -= vy` |
| mul-/2addr vx, vy | 乘法赋值 | `vx *= vy` |
| div-/2addr vx, vy | 除法赋值 | `vx /= vy` |
| rem-/2addr vx, vy | 取余赋值 | `vx %= vy` |
| 位运算 |  |  |
| and-/2addr vx, vy | 与赋值 | `vx &= vy` |
| or-/2addr vx, vy | 或赋值 | `vx \|= vy` |
| xor-/2addr vx, vy | 异或赋值 | `vx ^= vy` |
| shl-/2addr vx, vy | 左移赋值 | `vx <<= vy` |
| shr-/2addr vx, vy | 算术右移赋值 | `vx >>= vy` |
| ushr-/2addr vx, vy | 逻辑右移赋值 | `vx >>>= vy` |

我们可以查看这段代码：

```java
int a = 5,
    b = 2;
a += b;
a -= b;
a *= b;
a /= b;
a %= b;
a &= b;
a |= b;
a ^= b;
a <<= b;
a >>= b;
a >>>= b;
```

可能会编译成：

```smali
const/4 v0, 0x5
const/4 v1, 0x2
add-int/2addr v0, v1
sub-int/2addr v0, v1
mul-int/2addr v0, v1
div-int/2addr v0, v1
rem-int/2addr v0, v1
and-int/2addr v0, v1
or-int/2addr v0, v1
xor-int/2addr v0, v1
shl-int/2addr v0, v1
shr-int/2addr v0, v1
ushr-int/2addr v0, v1
```

### 一元运算

| 指令 | 运算类型 | 含义 |
| --- | --- | --- |
| 算术运算 |  |  |
| neg- vx, vy | 取负 | `vx = -vy` |
| 位运算 |  |  |
| not- vx, vy | 取补 | `vx = ~vy` |

简单来说，如果代码为`int a = 5, b = -a, c = ~a`，并且变量依次分配给`v0, v1, v2`的话，我们会得到`const/4 v0, 0x5`、`neg-int v1, v0`和`not-int v2, v0`。

## 跳转

### 无条件

Java 里面没有`goto`，但是 Smali 里面有，一般来说和`if`以及`for`配合的可能性很大，还有一个作用就是用于代码混淆。

| 指令 | 类型 |
| --- | --- |
| goto target | 8 位无条件跳 |
| goto/16 target | 16 位无条件跳 |
| goto/32 target | 32 位无条件跳 |

`target`在 Smali 中是标签，以冒号开头，使用方式是这样：

```smali
goto :label

# 一些语句

:label
```

这三个指令在使用形式上都一样，就是位数越大的语句支持的距离也越长。

### 条件跳转

`if`系列指令可用于`int`（以及`short`、`char`、`byte`、`boolean`甚至是对象引用）：

| 指令 | 含义 |
| --- | --- |
| if-eq vx,vy,target | `vx == vy`则跳到 target |
| if-ne vx,vy,target | `vx != vy`则跳到 target |
| if-lt vx,vy,target | `vx < vy`则跳到 target |
| if-ge vx,vy,target | `vx >= vy`则跳到 target |
| if-gt vx,vy,target | `vx > vy`则跳到 target |
| if-le vx,vy,target | `vx <= vy`则跳到 target |
| if-eqz vx,target | `vx == 0`则跳到 target |
| if-nez vx,target | `vx != 0`则跳到 target |
| if-ltz vx,target | `vx < 0`则跳到 target |
| if-gez vx,target | `vx >= 0`则跳到 target |
| if-gtz vx,target | `vx > 0`则跳到 target |
| if-lez vx,target | `vx <= 0`则跳到 target |

看一下这段代码：

```java
int a = 10
if(a > 0) 
    a = 1;
else
    a = 0;
```

可能的编译结果是：

```smali
const/4 v0, 0xa
if-lez v0, :cond_0 # if 块开始
const/4 v0, 0x1
goto :cond_1       # if 块结束
:cond_0            # else 块开始
const/4 v0, 0x0
:cond_1            # else 块结束
```

我们会看到用于比较逻辑是反着的，Java 里是大于，Smali 中就变成了小于等于，这个要注意。也有一些情况下，逻辑不是反着的，但是`if`块和`else`块会对调。还有，标签不一定是一样的，后面的数字会变，但是多数情况下都是两个标签，一个相对跳一个绝对跳。

如果只有`if`：

```java
int a = 10;
if(a > 0) 
    a = 1;
```

相对来说就简单一些，只需要在条件不满足时跳过`if`块即可：

```smali
int a = 10;
if(a > 0) 
    a = 1;
```

### 比较

对于`long`、`float`和`double`又该如何比较呢？Dalvik 提供了下面这些指令：

| 指令 | 含义 |
| --- | --- |
| cmpl-float vx, vy, vz | `vx = -sgn(vy - vz)` |
| cmpg-float vx, vy, vz | `vx = sgn(vy - vz)` |
| cmp-float vx, vy, vz | `cmpg-float`的别名 |
| cmpl-double vx, vy, vz | `vx = -sgn(vy - vz)` |
| cmpg-double vx, vy, vz | `vx = sgn(vy - vz)` |
| cmp-double vx, vy, vz | `cmpg-double`的别名 |
| cmp-long vx, vy, vz | `vx = sgn(vy - vz)` |

其中`sgn(x)`是符号函数，定义为：`x > 0`时值为 1，`x = 0`时值为 0，`x < 0`时值为 -1。

我们把之前例子中的`int`改为`float`：

```java
float a = 10;
if(a > 0) 
    a = 1;
else
    a = 0;
```

我们会得到：

```smali
const v0, 0x41200000 # float 10
const v1, 0x0
cmp-float v2, v0, v1
if-lez v2, :cond_0   # if 块开始
const v0, 0x3f800000 # float 1
goto :goto_0         # if 块结束
:cond_0              # else 块开始
const/4 v0, 0x0
:goto_0              # else 块结束
```

由于`cmpg`更类似平时使用的比较器，用起来更加顺手，但是`cmpl`也需要了解。

### `switch`

Dalvik 共支持两种`switch`，密集和稀疏。先来看密集`switch`，密集的意思是`case`的序号是挨着的：

```java
int a = 10;
switch (a){
    case 0:
        a = 1;
        break;
    case 1:
        a = 5;
        break;
    case 2:
        a = 10;
        break;
    case 3:
        a = 20;
        break;
}
```

编译为：

```smali
const/16 v0, 0xa

packed-switch v0, :pswitch_data_0 # switch 开始

:pswitch_0                        # case 0
const/4 v0, 0x1
goto :goto_0

:pswitch_1                        # case 1
const/4 v0, 0x5
goto :goto_0

:pswitch_2                        # case 2
const/16 v0, 0xa
goto :goto_0

:pswitch_3                        # case 3
const/16 v0, 0x14
goto :goto_0

:goto_0                           # switch 结束
return-void

:pswitch_data_0                   # 跳转表开始
.packed-switch 0x0                # 从 0 开始
    :pswitch_0
    :pswitch_1
    :pswitch_2
    :pswitch_3
.end packed-switch                # 跳转表结束
```

然后是稀疏`switch`：

```java
int a = 10;
switch (a){
    case 0:
        a = 1;
        break;
    case 10:
        a = 5;
        break;
    case 20:
        a = 10;
        break;
    case 30:
        a = 20;
        break;
}
```

编译为：

```smali
const/16 v0, 0xa

sparse-switch v0, :sswitch_data_0 # switch 开始

:sswitch_0                        # case 0
const/4 v0, 0x1
goto :goto_0

:sswitch_1                        # case 10
const/4 v0, 0x5

goto :goto_0

:sswitch_2                        # case 20
const/16 v0, 0xa
goto :goto_0

:sswitch_3                        # case 15
const/16 v0, 0x14
goto :goto_0

:goto_0                           # switch 结束
return-void

.line 55
:sswitch_data_0                   # 跳转表开始
.sparse-switch
    0x0 -> :sswitch_0
    0xa -> :sswitch_1
    0x14 -> :sswitch_2
    0x1e -> :sswitch_3
.end sparse-switch                # 跳转表结束
```

## 数组操作

数组拥有一套特化的指令。

### 创建

| 指令 | 含义 |
| --- | --- |
| new-array vx,vy,type | 创建类型为`type`，大小为`vy`的数组赋给`vx` |
| filled-new-array {params},type\_id | 从`params`创建数组，结果使用`move-result`获取 |
| filled-new-array-range {vx..vy},type\_id | 从`vx`与`vy`之间（包含）的所有寄存器创建数组，结果使用`move-result`获取 |

对于第一条指令，如果我们这样写：

```java
int[] arr = new int[10];
```

就可以使用该指令编译：

```smali
const/4 v1, 0xa
new-array v0, v1, I
```

但如果我们直接使用数组字面值给一个数组赋值：

```java
int[] arr = {1, 2, 3, 4, 5};
// 或者
arr = new int[]{1, 2, 3, 4, 5};
```

可以使用第二条指令编写如下：

```smali
const/4 v1, 0x1
const/4 v2, 0x2
const/4 v3, 0x3
const/4 v4, 0x4
const/4 v5, 0x5
filled-new-array {v1, v2, v3, v4, v5}, I
move-result v0
```

我们这里的寄存器是连续的，实际上不一定是这样，如果寄存器是连续的，还可以改写为第三条指令：

```smali
const/4 v1, 0x1
const/4 v2, 0x2
const/4 v3, 0x3
const/4 v4, 0x4
const/4 v5, 0x5
filled-new-array-range {v1..v5}, I
move-result v0
```

### 元素操作

`aget`系列指令用于读取数组元素，效果为`vx = vy[vz]`：

```smali
aget vx,vy,vz
aget-wide vx,vy,vz
aget-object vx,vy,vz
aget-boolean vx,vy,vz
aget-byte vx,vy,vz
aget-char vx,vy,vz
aget-short vx,vy,vz
```

有两个指令需要说明，`aget`用于获取`int`和`float`，`aget-wide`用于获取`long`和`double`。

同样，`aput`系列指令用于写入数组元素，效果为`vy[vz] = vx`：

```smali
aget vx,vy,vz
aget-wide vx,vy,vz
aget-object vx,vy,vz
aget-boolean vx,vy,vz
aget-byte vx,vy,vz
aget-char vx,vy,vz
aget-short vx,vy,vz
```

如果我们编写以下代码：

```java
int[] arr = new int[2];
int b = arr[0];
arr[1] = b;
```

可能会编译成：

```smali
const/4 v0, 0x2
new-array v1, v0, I
const/4 v0, 0x0
aget-int v2, v1, v0
const/4 v0, 0x1
aput-int v2, v1, v0
```

## 对象操作

### 对象创建

| 指令 | 含义 |
| --- | --- |
| new-instance vx, type | 创建`type`的新实例，并赋给`vx` |

`new-instance`用于创建实例，但之后还需要调用构造器`<init>`，比如：

```java
Object obj = new Object();
```

会编译成：

```smali
new-instance v0, Ljava/lang/Object;
invoke-direct-empty {v0}, Ljava/lang/Object;-><init>()V
```

方法调用后面再讲。

### 字段操作

`sget`系列指令用于获取静态字段，效果为`vx = class.field`：

```smali
sget vx, type->field:field_type
sget-wide vx, type->field:field_type
sget-object vx, type->field:field_type
sget-boolean vx, type->field:field_type
sget-byte vx, type->field:field_type
sget-char vx, type->field:field_type
sget-short vx, type->field:field_type
```

`sput`系列指令用于设置静态字段，效果为`class.field = vx`：

```smali
sput vx, type->field:field_type
sput-wide vx, type->field:field_type
sput-object vx, type->field:field_type
sput-boolean vx, type->field:field_type
sput-byte vx, type->field:field_type
sput-char vx, type->field:field_type
sput-short vx, type->field:field_type
```

我们在这里创建一个类：

```java
public class Test 
{
    private static int staticField;

    public static int getStaticField() {
        return staticField;
    }

    public static void setStaticField(int staticField) {
        Test.staticField = staticField;
    }
}
```

编译之后，我们可以在`getStaticField`中找到：

```smali
sget v0, Lnet/flygon/myapplication/Test;->staticField:I
return v0
```

在`setStaticField`中可以找到：

```smali
sput p0, Lnet/flygon/myapplication/Test;->staticField:I
return-void
```

`iget`系列指令用于获取实例字段，效果为`vx = vy.field`：

```smali
iget vx, vy, type->field:field_type
iget-wide vx, vy, type->field:field_type
iget-object vx, vy, type->field:field_type
iget-boolean vx, vy, type->field:field_type
iget-byte vx, vy, type->field:field_type
iget-char vx, vy, type->field:field_type
iget-short vx, vy, type->field:field_type
```

`iput`系列指令用于设置实例字段，效果为`vy.field = vx`：

```smali
iput vx, vy, type->field:field_type
iput-wide vx, vy, type->field:field_type
iput-object vx, vy, type->field:field_type
iput-boolean vx, vy, type->field:field_type
iput-byte vx, vy, type->field:field_type
iput-char vx, vy, type->field:field_type
iput-short vx, vy, type->field:field_type
```

我们将之前的类修改一下：

```smali
public class Test
{
    private int instanceField;

    public int getInstanceField() {
        return instanceField;
    }

    public void setInstanceField(int instanceField) {
       this.instanceField = instanceField;
    }
}
```

反编译之后，我们可以在`getInstanceField`中找到：·

```smali
iget v0, p0, Lnet/flygon/myapplication/Test;->instanceField:I
return v0
```

在`setInstanceField`中可以找到：

```smali
iset p1, p0, Lnet/flygon/myapplication/Test;->instanceField:I
return-void
```

在实例方法中，`this`引用永远是`p0`。第一个参数从`p1`开始。

### 方法调用

有五类方法调用指令：

| 指令 | 含义 |
| --- | --- |
| invoke-static | 调用静态方法 |
| invoke-direct | 调用直接方法 |
| invoke-direct-empty | 无参的`invoke-direct` |
| invoke-virtual | 调用虚方法 |
| invoke-super | 调用超类的虚方法 |
| invoke-interface | 调用接口方法 |

这些指令的格式均为：

```smali
invoke-* {params}, type->method(params_type)return_type
```

如果需要传递`this`引用，将其放置在`param`的第一个位置。

那么这些指令有什么不同呢？首先要分辨两个概念，虚方法和直接方法（JVM 里面叫特殊方法）。其实 Java 是没有虚方法这个概念的，但是 DVM 里面有，直接方法是指类的（`type`为某个类）所有实例构造器和`private`实例方法。反之`protected`或者`public`方法都叫做虚方法。

`invoke-static`比较好分辨，当且仅当调用静态方法时，才会使用它。

`invoke-direct`（在 JVM 中叫做`invokespecial`）用于调用直接方法，`invoke-virtual`用于调用虚方法。除了一种情况，显式使用`super`调用超类的虚方法时，使用`invoke-super`（直接方法仍然使用`invoke-direct`）。

就比如说，每个`Activity`的`onCreate`中要调用`super.onCreate`，该方法属于虚方法，于是我们会看到：

```smali
invoke-super {p0, p1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V
```

但是呢，每个`Activity`构造器里面要调用`super`的无参构造器，它属于直接方法，那么我们会看到：

```smali
invoke-direct {p0}, Landroid/app/Activity;-><init>()V
```

`invoke-interface`用于调用接口方法，接口方法就是接口的方法，`type`一定为某个接口，而不是类。换句话说，类中实现的方法仍然是虚方法。比如我们在某个对象上调用`Map.get`，属于接口方法，但是调用`HashMap.get`，属于虚方法。这个指令一般在向上转型为接口类型的时候出现。

此外，五类指令中每一个都有对应的`invoke-*-range`指令，格式为：

```smali
invoke-*-range {vx..vy},type->method(params_type)return_type
```

如果参数所在的寄存器的连续的，可以替换为这条指令。

### 对象转换

对象转换有自己的一套检测方式，DVM 使用以下指令来实现：

| 指令 | 含义 |
| --- | --- |
| instance-of vx, vy, type | 检验`vy`的类型是不是`type`，将结果存入`vx` |
| check-cast vx, type | 检验`vx`类型是不是`type`，不是的话会抛出`ClassCastException` |

`instance-of`指令对应 Java 的`instanceof`运算符。如果我们编写：

```java
String s = "test";
boolean b = s instanceof String;
```

可能会编译为：

```smali
const-string v0, "test"
instance-of v1, v0, Ljava/lang/String;
```

`check-cast`用于对象类型强制转换的情况，如果我们编写：

```java
String s = "test";
Object o = (Object)s;
```

那么就会：

```java
const-string v0, "test"
check-cast v0, Ljava/lang/Object;
move-object v1, v0
```

## 返回

```smali
return-void
return vx
return-wide vx
return-object vx
```

如果函数无返回值，那么使用`return-void`，注意在 Java 中，无返回值函数结尾处的`return`可以省，而 Smali 不可以。

如果函数需要返回对象，使用`return-object`；需要返回`long`或者`double`，使用`return-wide`；除此之外所有情况都使用`return`。

## 异常指令

异常指令实际上只有一条，但是代码结构相当复杂。

我们需要看看 Smali 如何处理异常。

### try-catch

不失一般性，我们构造以下语句：

```java
int a = 10;
try {
    callSomeMethod();
} finally {
    a = 0;
}
callAnotherMethod();
```

可能会编译成这样，这些语句每个都不一样，可以按照特征来定位：

```smali
const/16 v0, 0xa

:try_start_0            # try 块开始
invoke-direct {p0}, Lnet/flygon/myapplication/SubActivity;->callSomeMethod()V
:try_end_0              # try 块结束

.catch Ljava/lang/Exception; {:try_start_0 .. :try_end_0} :catch_0

:goto_0
invoke-direct {p0}, Lnet/flygon/myapplication/SubActivity;->callAnotherMethod()V
return-void

:catch_0                # catch 块开始
move-exception v1
const/4 v0, 0x0
goto :goto_0            # catch 块结束
```

我们可以看到，`:try_start_0`和`:try_end_0`之间的语句如果存在异常，则会向下寻找`.catch`（或者`.catch-all`）语句，符合条件时跳到标签的位置，这里是`:catch_0`，结束之后会有个`goto`跳回去。

### try-finally

```java
int a = 10;
try {
    callSomeMethod();
} catch (Exception e) {
    a = 1;
}
finally {
    a = 0;
}
callAnotherMethod();
```

编译之后是这样：

```smali
const/16 v0, 0xa

:try_start_0            # try 块开始
invoke-direct {p0}, Lnet/flygon/myapplication/SubActivity;->callSomeMethod()V
:try_end_0              # try 块结束

.catchall {:try_start_0 .. :try_end_0} :catchall_0

const/4 v0, 0x0         # 复制一份到外面
invoke-direct {p0}, Lnet/flygon/myapplication/SubActivity;->callAnotherMethod()V
return-void

:catchall_0             # finally 块开始
move-exception v1
const/4 v0, 0x0
throw v1                # finally 块结束
```

我们可以看到，编译器把`finally`编译成了重新抛出的`.catch-all`，这在逻辑上也是说得通的。但是，`finally`中的逻辑在无异常情况下也会执行，所以需要复制一份到`finally`块的后面。

### try-catch-finally

下面看看如果把这两个叠加起来会怎么样。

```java
int a = 10;
try {
    callSomeMethod();
} catch (Exception e) {
    a = 1;
}
finally {
    a = 0;
}
callAnotherMethod();
```

```smali
const/16 v0, 0xa

:try_start_0            # try 块开始
invoke-direct {p0}, Lnet/flygon/myapplication/SubActivity;->callSomeMethod()V
:try_end_0              # try 块结束

.catch Ljava/lang/Exception; {:try_start_0 .. :try_end_0} :catch_0
.catchall {:try_start_0 .. :try_end_0} :catchall_0

const/4 v0, 0x0         # 复制一份到外面

:goto_0
invoke-direct {p0}, Lnet/flygon/myapplication/SubActivity;->callAnotherMethod()V
return-void

:catch_0                # catch 块开始
move-exception v1
const/4 v0, 0x1
const/4 v0, 0x0         # 复制一份到 catch 块里面
goto :goto_0            # catch 块结束

:catchall_0             # finally 块开始
move-exception v2
const/4 v0, 0x0
throw v2                # finally 块结束
```

我们可以看到，其中同时含有`.catch`块和`.catchall`块。有一些不同之处在于，`finally`块中的语句异常发生时也要执行，并且如果把`finally`编译成`.catchall`，那么和`.catch`就是互斥的，所以要复制一份到`catch`块里面。特别是`finally`块中的语句一多，就容易乱。

## 锁

| 指令 | 含义 |
| --- | --- |
| monitor-enter vx | 获得`vx`所引用的对象的锁 |
| monitor-exit vx | 释放`vx`所引用的对象的锁 |

对应 Java 的`synchronized`语句。而`synchronized`一般是被`try-finally`包起来的。

如果你编写：

```java
int a = 1;
synchronized(this) {
    a = 2;
}
```

就相当于

```java
int a = 1;
// monitor-enter this
try {
    a = 2;
} finally {
    // monitor-exit this
}
```

此外 Java 中没有与这两条指令相对应的方法，所以这两条指令一定成对出现。

## 数据转换

### 整数与浮点以及浮点与浮点

```smali
int-to-float vx, vy
int-to-double vx, vy
long-to-float vx, vy
long-to-double vx, vy
float-to-int vx, vy
float-to-long vx,vy
float-to-double vx, vy
double-to-int vx, vy
double-to-long vx, vy
double-to-float vx, vy
```

因为它们的表示方式不同，所以要保持表示的值不变，重新计算二进制位。如果不转换的话，就相当于二进制位不变，而表示的值改变，结果毫无意义。比如前面的`0.1f`如果不转换为直接使用，就会表示`0x3dcccccd`。

### 整数之间的向上转换

这种转换方式相当直接，`int`向`long`转换，`long`的第一个寄存器完全复制，第二个寄存器以`int`的最高位填充。除此之外没有其它的指令了，因为比`int`小的整数其实都是 32 位表示的，只是有效范围是 8 位或 16 位罢了（见数据定义）。

```smali
int-to-long vx,vy
```

### 整数之间的向下转换

其规则是数据位截断，符号位保留。每个整数的最高位都是符号位，其余是数据位。以`int`转`short`为例，`int`的低 15 位复制给`short`，然后`int`的最高位（符号位）复制给`short`的最高位。其它同理。如果不转换而直接使用的话，会直接截断低 16 位，符号可能不能保留。

```smali
long-to-int vx,vy
int-to-byte vx,vy
int-to-char vx,vy
int-to-short vx,vy
```

## NOP

`nop`指令表示无操作。在一些场合下，不能修改二进制代码的字节数和偏移，需要用`nop`来填充，但是安卓逆向中几乎用不到。

## 参考

- [Bytecode for the Dalvik VM](http://www.netmite.com/android/mydroid/dalvik/docs/dalvik-bytecode.html)
- [Dalvik字节码含义查询表](http://blog.csdn.net/jiayanhui2877/article/details/41008985)
- [DVM 指令集图解](https://github.com/corkami/pics/blob/master/binary/DVM.png)

> 作者：[飞龙](https://github.com/wizardforcel)
