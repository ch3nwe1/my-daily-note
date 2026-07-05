## abs
**计算并返回一个数字的绝对值（Absolute Value）**。
**整数**
```python
print(abs(-10))  # 输出: 10
print(abs(5))    # 输出: 5
```
**浮点数**
```python
print(abs(-3.14)) # 输出: 3.14
print(abs(0.0))   # 输出: 0.0
```
复数
当传入一个复数（如 $a + bj$）时，`abs()` 返回的不是简单的去负号，而是这个复数的**模（Magnitude）**，即该点到坐标原点的几何距离。
```python
# 在 Python 中，复数的虚部用 j 表示
c = 3 + 4j

# 计算 3 和 4 的勾股弦长：√(3² + 4²) = √(9 + 16) = √25 = 5.0
print(abs(c))  # 输出: 5.0 (复数的模永远返回 float 类型)
```
**魔术方法**
```python
import math

class Vector2D:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    # 重载 abs() 运算符
    def __abs__(self):
        print("--> 触发了 __abs__ 魔术方法")
        # 返回向量的模长（几何距离）
        return math.sqrt(self.x**2 + self.y**2)

v = Vector2D(6, 8)
print(abs(v))  
# 输出:
# --> 触发了 __abs__ 魔术方法
# 10.0
```

## all(_iterable_, _/_)
如果 _iterable_ 的所有元素均为真值（或可迭代对象为空）则返回 `True`。等价于:
```python
def all(iterable):
    for element in iterable:
        if not element:
            return False
    return True
```
```python
# 全为 True
print(all([True, 1, "hello"]))  # 输出: True

# 包含一个 False
print(all([True, 0, "hello"]))  # 输出: False (因为 0 的布尔值是 False)

# 包含一个空字符串
print(all(["apple", "", "banana"]))  # 输出: False (因为 "" 是假值)

# 传入空的可迭代对象
print(all([]))  # 输出: True
print(all({}))  # 输出: True
```
当对字典使用 `all()` 时，Python 默认检查的是字典的 **Key（键）**，而不是 Value（值）
```python
# 键都是非零/非空，所以是 True
my_dict = {1: False, 2: False} 
print(all(my_dict))  # 输出: True

# 有一个键是 0 (False)
bad_dict = {0: "hello", 1: "world"}
print(all(bad_dict))  # 输出: False
```
## any(_iterable_, _/_)
如果 _iterable_ 的任一元素为真值则返回 `True`。如果可迭代对象为空，返回 `False`。等价于:
```python
def any(iterable):
    for element in iterable:
        if element:
            return True
    return False
```
## bool
在 Python 中，`bool`（布尔）类型用来表示**真**或**假**。它只有两个常量值：**`True`**（真）和 **`False`**（假）。
如果你把以下内容传给 `bool()` 函数，它们都会返回 `False`

| 分类  |        假值        |     说明     |
| :-: | :--------------: | :--------: |
| 常量  |    None,False    |  内置的空值和假值  |
| 数值零 |    0，0.0， 0j     |  任何代表零的数字  |
| 空序列 | "",[],(),rang(0) | len为0的内置序列 |
| 空映射 |        {}        | 没有任何元素的容器  |
## @classmethod
把一个方法封装成类方法。
类方法隐含的第一个参数就是类，就像实例方法接收实例作为参数一样
```python
class C:
    @classmethod
    def f(cls, arg1, arg2): ...
```
## delattr 与 setattr
`setattr()` 用于给对象设置属性。如果属性已存在，则**修改**它的值；如果属性不存在，则**自动创建**
```python
setattr(object, name, value)
```
```python
class User:
    def __init__(self, name):
        self.name = name

user = User("Alice")

# 1. 修改已有的属性
setattr(user, "name", "Bob")
print(user.name)  # 输出: Bob

# 2. 动态添加原本不存在的属性
setattr(user, "age", 25)
print(user.age)   # 输出: 25

# 相当于：user.age = 25，但这里的 "age" 可以是一个变量
```
`delattr()` 用于从对象中删除指定的属性。如果尝试删除一个不存在的属性，程序会抛出 `AttributeError` 异常。
```python
delattr(object, name)
```
```python
class Product:
    def __init__(self, name, price):
        self.name = name
        self.price = price

gadget = Product("Phone", 999)

# 删除 price 属性
delattr(gadget, "price")

# 尝试访问已被删除的属性
try:
    print(gadget.price)
except AttributeError as e:
    print(e)  # 输出: 'Product' object has no attribute 'price'

# 相当于：del gadget.price，但允许使用字符串变量
```
为什么不直接用 `obj.attr`？
通常我们用 `obj.attr = value` 和 `del obj.attr` 来操作属性，那为什么还要用这两个内置函数呢？
核心区别：点语法是静态的，而函数是动态的
## divmod
接受两个（非复数）数字作为参数并返回由当对其使用整数除法时的商和余数组成的数字对
```python
divmod(5,2)
(2,1)
```

## eval
它的功能是：**将一个字符串当作 Python 表达式来执行，并返回执行结果。**
由于安全隐患太大，开发中**强烈建议不要使用 `eval()`**。针对它的常见应用场景，Python 提供了更安全的替代方案
`ast.literal_eval` 属于 Python 自带的 `ast` 模块。它**只会**解析单纯的字面量（如数字、字符串、列表、字典、元组、布尔值和 None），如果字符串里包含任何函数调用或恶意命令，它会直接报错拒绝执行。
## filter(_function_, _iterable_, _/_)
使用 _iterable_ 中 _function_ 返回真值的元素构造一个迭代器。 _iterable_ 可以是一个序列，一个支持迭代的容器或者一个迭代器。如果 _function_ 为 `None`，则会使用标识函数，也就是说，_iterable_ 中所有具有假值的元素都将被移除。
请注意，`filter(function, iterable)` 相当于一个生成器表达式，当 function 不是 `None` 的时候为 `(item for item in iterable if function(item))`；function 是 `None` 的时候为 `(item for item in iterable if item)`.
```python
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# 过滤规则：n % 2 == 0 结果为 True 的保留
result = filter(lambda n: n % 2 == 0, numbers)

print(list(result))  # 输出: [2, 4, 6, 8, 10]
```
```python
data = [1, 0, "Hello", "", False, [1, 2], None, True]

# 传入 None，只保留真值
clean_data = filter(None, data)

print(list(clean_data))  # 输出: [1, 'Hello', [1, 2], True]
```
## frozenset
返回一个新的 frozenset 对象，它包含可选参数 iterable 中的元素。frozenset 是一个内置的类。有关此类的文档

## @staticmethod
将方法转换为静态方法。

静态方法不会接收隐式的第一个参数。要声明一个静态方法，请使用此语法
```python
class C:
    @staticmethod
    def f(arg1, arg2, argN): ...
```