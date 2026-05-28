> 主要内置类型有数字、序列、映射、类、实例和异常。

## 逻辑值检测

> 在默认情况下，一个对象会被视为具有真值，除非其所属的类在该对象上调用时，定义了 __bool__() 方法返回 False 或 __len__() 方法返回零。如果其中某一方法被调用时引发了异常，该异常将被传播并且该对象将不具有真值 (例如 NotImplemented)。

```python
class A:...  
a = A()  
if a:  
    print('object is True')
```

实现了`__bool__`和`__len__`

```python
  
class A(list):  
  
	# 会调用
    def __bool__(self):  
        print("调用了__bool__")  
        return False  
	# __bool__存在不会调用__len__
    def __len__(self):  
        print('调用了__len__')  
        return NotImplemented  
  
a = A()  
if a:  
    print('object is True')
    
# 调用了__bool__
```

以下基本完整地列出了具有假值的内置对象
-  被定义为假值的常量: `None` 和 `False`
- 任何数值类型的零: `0`, `0.0`, `0j`, `Decimal(0)`, `Fraction(0, 1)`
- 空的序列和多项集: `''`, `()`, `[]`, `{}`, `set()`, `range(0)`

## 布尔运算 --- and, or, not

这些属于布尔运算，按优先级升序排列：

|    运算     |                结果                |                                       备注                                        |
| :-------: | :------------------------------: | :-----------------------------------------------------------------------------: |
| `x or y`  |     如果 _x_ 为真值，则 _x_，否则 _y_      |                       这是个短路运算符，因此只有在第一个参数为假值时才会对第二个参数求值。                        |
| `x and y` |   如果 _x_ 为假值，则返回 _x_，否则返回 _y_    |                       这是个短路运算符，因此只有在第一个参数为真值时才会对第二个参数求值。                        |
|  `not x`  | 如果 _x_ 为假值，则为 `True`，否则为 `False` | `not` 的优先级比非布尔运算符低，因此 `not a == b` 会被解读为 `not (a == b)` 而 `a == not b` 会引发语法错误。 |
```python
x or y and z
# 这里说的and优先级比or高，我来解释一下
# 首先python是从左到右解析，如果x为False，直接返回False，这是短路原则。
# 如果x为True，则解析 y and z的结果再与x比较，而不是 x和y比较。所以and比or优先级高
```

## 比较运算符
在 Python 中有八种比较运算符。它们的优先级相同（比布尔运算的优先级高）

|运算|含意|
|---|---|
|`<`|严格小于|
|`<=`|小于或等于|
|`>`|严格大于|
|`>=`|大于或等于|
|`==`|等于|
|`!=`|不等于|
|`is`|对象标识|
|`is not`|否定的对象标识|
## 数字类型 --- int,float,complex
存在三种不同的数字类型：_整数_, _浮点数_ 和 _复数_。此外，布尔值属于整数的子类型。整数具有无限的精度。
数字是由数字字面值或内置函数与运算符的结果来创建的。不带修饰的整数字面值（包括十六进制、八进制和二进制数）会生成整数。 包含小数点或幂运算符的数字字面值会生成浮点数

构造函数 [`int()`](https://docs.python.org/zh-cn/3.14/library/functions.html#int "int")、 [`float()`](https://docs.python.org/zh-cn/3.14/library/functions.html#float "float") 和 [`complex()`](https://docs.python.org/zh-cn/3.14/library/functions.html#complex "complex") 可以用来构造特定类型的数字。

Python 完整支持混合算术运算：当一个双目算术运算符的操作数具有不同的内置数字类型时，“较窄类型”的操作数会被加宽为与另一个操作数的类型
-  如果两个参数均为复数，则不会执行任何转换；
- 如果任一参数为复数或浮点数，另一参数将被转换为浮点数；
- 否则，两者应该都为整数，不需要进行转换。
```python
1+1.0
# 2.0
complex(1,2) + 1
# 2+2j
```

## 布尔类型 - bool
代表真值的布尔对象。 bool 类型只有两个常量实例: `True` 和 `False`。
bool 是 int 的子类 。 在许多数字场景下，False 和 True 的行为分别与整数 0 和 1 类似。但是，不建议这样使用；请使用 int() 显式地执行转换。

## 迭代器类型

在 Python 中，协议（Protocol）通常指一种**鸭子类型（Duck Typing）的接口规范**——不要求你继承特定的父类，只要你实现了特定的方法，你就是这个角色。

**迭代器协议**非常简单，它只要求一个对象必须实现以下两个方法
1. `__iter__()`:
	让迭代器对象自身也是“可迭代的”（Iterable），这样它就能直接放进 `for` 循环中。
2. `__next__()`
		返回容器的下一个元素.如果没有更多元素了，它必须抛出 **`StopIteration`** 异常。`for` 循环捕捉到这个异常后，就会漂亮地优雅收尾（终止循环）。
```python
  
class CountDown:  
    def __init__(self, n):  
        self.current = n  
  
    def __iter__(self):  
        return self  
  
    def __next__(self):  
        if self.current <= 0:  
            raise StopIteration  
        result = self.current  
        self.current -= 1  
        return result  
  
cd = CountDown(3)  
for i in cd:  
    print(i)

# 3
# 2
# 1

```

## 生成器对象
任何包含了 **`yield`** 关键字的 Python 函数都被称为生成器函数。当你调用这个函数时，它**不会立即执行函数体内的代码**，而是返回一个特殊的**生成器对象（generator object）**

**工作原理**：每次对该对象调用 `next()`，函数就会执行到下一个 `yield` 语句处，将 `yield` 后面的值返回，然后“冻结”在当前位置。

```python
  
def count_down(count):  
    print('计数开始')  
    while count > 0:  
        yield count  
        count -= 1  
    print('计数结束')  
  
  
g = count_down(5)  
print(type(g)) # <class 'generator'>  
  
print(next(g))  
# 计数开始  
# 5  
print(next(g))  
# 4  
print(next(g))  
# 3  
print(next(g))  
# 2  
print(next(g))  
# 1  
print(next(g))  
# 计数结束  
# raise StopIteration	
```
## 序列类型 --- list,tuple,range

**通用序列操作**

|运算|结果：|备注|
|---|---|---|
|`x in s`|如果 _s_ 中的某项等于 _x_ 则结果为 `True`，否则为 `False`|(1)|
|`x not in s`|如果 _s_ 中的某项等于 _x_ 则结果为 `False`，否则为 `True`|(1)|
|`s + t`|_s_ 与 _t_ 相拼接|(6)(7)|
|`s * n` 或 `n * s`|相当于 _s_ 与自身进行 _n_ 次拼接|(2)(7)|
|`s[i]`|_s_ 的第 _i_ 项，起始为 0|(3)(8)|
|`s[i:j]`|_s_ 从 _i_ 到 _j_ 的切片|(3)(4)|
|`s[i:j:k]`|_s_ 从 _i_ 到 _j_ 步长为 _k_ 的切片|(3)(5)|
|`len(s)`|_s_ 的长度||
|`min(s)`|_s_ 的最小项||
|`max(s)`|_s_ 的最大项||
(1)
```python
"gg" in "eggs"
True
```
(2)小于 `0` 的 _n_ 值会被当作 `0` 来处理 (生成一个与 _s_ 同类型的空序列)。请注意序列 _s_ 中的项并不会被拷贝；它们会被多次引用。这一点经常会令 Python 编程新手感到困扰；例如:
```python
lists = [[]] * 3
[[],[],[]]
lists[0].append(1)
[[1],[1],[1]]
```
(3)如果 _i_ 或 _j_ 为负值，则索引顺序是相对于序列 _s_ 的末尾：索引号会被替换为 `len(s) + i` 或 `len(s) + j`。但要注意 `-0` 仍然为 `0`。

**序列方法**
`count(_value_, _/_)`
> 返回 _value_ 在 _sequence_ 中出现的总次
`sequence.index(_value_[, _start_[, _stop_]])`
> 返回 _value_ 在 _sequence_ 中首次出现所在的索引号。
