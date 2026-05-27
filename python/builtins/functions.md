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