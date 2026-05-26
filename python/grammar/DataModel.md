## None
`None`是一个特殊的常量,用于表示空值,没有值或不存在的对象,类型是`NoneType`
```python
>>> a=None
>>> type(a)
<class 'NoneType'>
>>> 
```
**常见的应用场景:**
1. 函数的默认返回值,如果一个函数没有显式的使用`return`语句,或者只写了`return`但后面没有任何东西,Python会默认返回`None`
```python
>>> def test():\
...     pass
...     
>>> print(test())
None
```
```python
>>> def test():\
...     return
...     
>>> print(test())
None
```
2. 变量初始化占位符:当你想定义一个变量但又不知道该赋具体什么值的时候,可以设置为`None`
```python
x = None
```
3. 可选参数的默认值,常用语函数定义中,表示某个参数是可选的
```python
def func(x=None):
	pass
```
**如何正确的判断`None`**
在Python中,推荐使用`is`和`is not` 来判断一个变量是否为`None`
```python
x = None

if x is None:
	print('x is None')

if x is not None:
	print('x is not None')
```
**None 的布尔值属性**
在条件判断中(`if`语句)中,`None`会被当作`Flase`来处理,但是它并不等于`False`,`0`或`[]`,他们在类型上是完全不同的
```python
x = None 
if not x:
	print('None as False')

print(x == False) # False
print(x == 0) # False
print(x == []) # False
```
## NotImplemented
在 Python 中，`NotImplemented` 是一个非常特殊的**内置常量**。它和我们熟知的 `None`、`True`、`False` 一样，属于单例对象（在内存中只有一个）。
`NotImplemented` 主要用于 Python 的**富比较方法**（如 `__eq__`, `__lt__`, `__gt__` 等）和**算术运算符方法**（如 `__add__`, `__sub__` 等）中。
当一个对象不知道如何与另一个对象进行计算或比较时，它会返回 `NotImplemented`。这相当于在告诉 Python：**“我不知道怎么处理这个操作，请问问对方能不能处理。”**

当你执行 `a + b` 时，Python 的幕后发生的事情如下：
- Python 首先调用 `a.__add__(b)`。    
- 如果 `a` 的类认得 `b` 的类型，它会返回计算结果。
- 如果 `a` 的类不认得 `b` 的类型，它就会返回 `NotImplemented`。
- **关键一步：** 一旦 Python 收到 `NotImplemented`，它不会立刻报错，而是会转向尝试调用 `b.__radd__(a)`（右加方法）。
- 如果 `b` 能够处理 `a`，则返回正确结果；如果 `b` 也返回了 `NotImplemented`，Python 这时才会抛出 `TypeError` 异常。
假设我们自己写了一个 `Vector`（向量）类，想让它支持和整数 `int` 相加：
```python
class Vector:  
    def __init__(self, x, y):  
        self.x = x  
        self.y = y  
  
    def __add__(self, other):  
        """ 自定义加法  """  
        if isinstance(other, Vector):  
            return Vector(self.x + other.x, self.y + other.y)  
  
        if isinstance(other, int):  
            return Vector(self.x + other, self.y + other)  
  
        return NotImplemented  
  
    def __radd__(self, other):  
        return self.__add__(other)  
  
    def __repr__(self):  
        return f"Vector({self.x}, {self.y})"  
  
  
v = Vector(1, 2)  
  
# Vector + int -> 触发 Vector.__add__方法  
print(v + 5) # Vector(6,7)  
  
# int + Vector  
# 触发 int.__add__方法,不认识Vector  
# 触发 Vector.__radd__方法  
print(5 + v) # Vector(6,7)  
  
print(v+ 'hello') # 两边都返回 NotImplemented，Python 抛出 TypeError
```
## Ellipsis

在 Python 中，`Ellipsis`（中文意为“省略号”）也是一个内置的**全局常量**，它的字面量简写形式是三个英文句号 `...`。
```python
print(Ellipsis)   # 输出: Ellipsis
print(...)        # 输出: Ellipsis
print(... is Ellipsis)  # 输出: True
```
应用场景
1. 作为代码占位符（替代 `pass`）
```python
def my_future_function():
    ...  # 意思是“这里以后会写代码”，效果等同于 pass

class MyEmptyClass:
    ...
```
2. 类型提示（Type Hinting）
代表任意长度的元组（Tuple）：
如果我们想注解一个只包含整数、但长度不限的元组，可以这样写：
```python
from typing import Tuple

# 意思是：这是一个元组，里面可以有任意多个 int
my_tuple: Tuple[int, ...] = (1, 2, 3, 4, 5)
```
代表任意参数的函数（Callable）：
```python
from typing import Callable

# 意思是：接受一个函数，该函数接收任意参数(...)，并返回一个字符串
def process_data(callback: Callable[..., str]):
    pass
```

##  numbers.Number

在 Python 中，`numbers.Number` 是所有**数值类型的抽象基类**

它的层级结构如下：
- **`Number`**：整个层级的根节点。所有数字类型都继承自它。
- **`Complex`**：复数（包含内置的 `complex`）。
- **`Real`**：实数（包含内置的 `float`）。
- **`Rational`**：有理数/分数（包含 `fractions.Fraction`）。
- **`Integral`**：整数（包含内置的 `int` 和 `bool`）。
## 序列
**不可变序列**
1. 字符串 str
2. 元组 tuple
3. 字节串 bytes
**可变序列**
4. 列表 list
5. 字节数组 bytearray
## 集合 set
1. 元素无序
2. 元素唯一
使用大括号<b>{}</b>创建
```python
fruits = {'apple','banana','cherry'}
print(fruits) #{'banana', 'apple', 'cherry'}
```
使用内置函数创建`set()`
创建空集合使用`set()`,使用`{}`构建的是空字典
```python
# 正确创建空集合  
empty_set = set()  
print(type(empty_set))  # 输出: <class 'set'>  
  
# 错误示范  
empty_dict = {}  
print(type(empty_dict))  # 输出: <class 'dict'>
```
使用其他可迭代对象转换为集合
```python
# 将列表(List)转换为集合（自动去重）
my_list = [1, 4, 4, 2, 5, 1]
my_set = set(my_list)
print(my_set)  # 输出: {1, 2, 4, 5}

# 将字符串(String)转换为集合（拆分为单个字符并去重）
char_set = set("hello")
print(char_set)  # 输出类似: {'h', 'e', 'l', 'o'}
```
创建不可变集合，使用`frozenset`

```python
# 创建不可变集合
frozen = frozenset([1, 2, 3, 3])
print(frozen)  # 输出: frozenset({1, 2, 3})

# 尝试添加元素会报错： AttributeError
# frozen.add(4)
```

什么对象可以放入集合？
Python 的集合有一个核心限制：**集合里的元素必须是“可哈希的”（Hashable），简单来说就是不可变对象**
```python
# 正确：元组是不可变的，可以作为集合元素
valid_set = {(1, 2), (3, 4)} 

# 错误：列表是可变的，会抛出 TypeError: unhashable type: 'list'
# invalid_set = {[1, 2], [3, 4]}
```