## 原型
### 属性引用
```ebnf
attributeref: primary "." identifier
```
产生过程可通过重载 `__getattribute__()` 方法或`__getattr__()` 方法来自定义。 将会先调用 `__getattribute__()` 方法并返回一个值或者如果属性不可用则会引发 `AttributeError`。

- `__getattribute__` **无条件触发**。只要你访问属性（如 `obj.x`），无论该属性存不存在，它都会首先被调用。
- `__getattr__`**只有在属性不存在时触发**。当 Python 在对象实例、类层级或父类中都找不到该属性时，才会调用它。
```python
class Demo:  
    def __init__(self):  
        self.exist_key = "我是存在的属性"  
  
    # 1. 替补：只有找不到属性时才触发  
    def __getattr__(self, item):  
        print(f"-> 进入 __getattr__: 找不到属性 [{item}]，由我来兜底")  
        return f"默认值_{item}"  
  
    # 2. 前锋：任何属性访问都会先进入这里  
    def __getattribute__(self, item):  
        print(f"-> 进入 __getattribute__: 正在尝试访问 [{item}]")  
        # 注意：这里必须调用父类的 __getattribute__，否则会导致死循环  
        return super().__getattribute__(item)  
  
# 实例化对象  
d = Demo()  
  
# 访问存在的属性  
print(d.exist_key)  
# 进入 __getattribute__: 正在尝试访问 [exist_key]# 我是存在的属性  
  
# 访问【不存在】的属性  
print(d.fake_key)  
# -> 进入 __getattribute__: 正在尝试访问 [fake_key]# -> 进入 __getattr__: 找不到属性 [fake_key]，由我来兜底  
# 默认值_fake_key
```

### 抽取与切片
**抽取** 语法通常被用于从 容器 中选择一个元素 -- 例如，从 dict 取一个值:
```python
digits_by_name = {'one': 1, 'two': 2}
digits_by_name['two']  # 使用键 'two' 进行对字典的抽取操作
```
在运行时，解释器将对原型和下标求值，并以下标作为参数调用原型的 `__getitem__()` 或 `__class_getitem__()` special method。 
- **`__getitem__`**：针对的是**实例对象**。用来让一个实例可以像列表或字典那样通过 `obj[key]` 取值。
- **`__class_getitem__`**：针对的是**类对象（Class）**。专门为了实现类似 `list[str]` 这样的类型提示（Type Hinting）而诞生的。

```python
class MyList:  
    def __init__(self, data):  
        self.data = data  
  
    def __getitem__(self, index):  
        print(f"正在访问实例的索引: {index}")  
        return self.data[index]  
  
obj = MyList([1, 2, 3])  
print(obj[1])  
  
# 正在访问实例的索引: 1  
# 2  
  
obj = MyList({"name":"张三","age":18})  
print(obj['name'])  
# 正在访问实例的索引: name  
# 张三
```

```python
  
class MyContainer:  
  
    def __class_getitem__(cls, item):  
        print(f"检测到类级别的方括号参数: {item}")  
        return cls  
  
# 注意：这里我们没有实例化对象，而是直接在【类名】后面加方括号！  
# 这种写法在类型标注中很常见，比如：x: MyContainer[int]  
result = MyContainer[int]  
  
# 检测到类级别的方括号参数: <class 'int'>
```
**切片**一种更高级的抽取形式 _切片_ 通常被用于从 [序列](https://docs.python.org/zh-cn/3.14/reference/datamodel.html#datamodel-sequences) 中提取一部分，在这种形式中，下标是一个 slice: 最多三个以冒号分隔的表达式。 其中任何表达式都可以省略，但切片必须包含至少一个冒号:
```python
number_names = ['zero', 'one', 'two', 'three', 'four', 'five']
number_names[1:3]
['one', 'two']
number_names[1:]
['one', 'two', 'three', 'four', 'five']
number_names[:3]
['zero', 'one', 'two']
number_names[:]
['zero', 'one', 'two', 'three', 'four', 'five']
number_names[::2]
['zero', 'two', 'four']
number_names[:-3]
['zero', 'one', 'two']
del number_names[4:]
number_names
['zero', 'one', 'two', 'three']
```

### 调用

在 Python 中，`__call__` 是一个非常酷的魔术方法（Magic Method）。它的核心作用是：**让一个类的“实例对象”可以像“函数”一样被直接调用**
通常情况下，我们只有面对函数或方法时才会写小括号 `obj()`。但如果一个类实现了 `__call__` 方法，那么这个类实例化出来的**对象**，也能直接加小括号执行。

在 Python 的术语中，实现了 `__call__` 的对象被称为 **“可调用对象”（Callable）**。

```python
  
class Counter:  
    def __init__(self):  
        self.count = 0  
  
    def __call__(self, *args):  
        self.count += 1  
        print(f'args: {args}')  
        return self.count  
  
f = Counter()  
  
print(f'result: {f(1)}')  
print(f'result: {f(2,3)}')  
print(f'result: {f([1,2,3])}')

# args: (1,)
# result: 1
# args: (2, 3)
# result: 2
# args: ([1, 2, 3],)
# result: 3
```

## await
在 Python 中，`await` 是异步编程（Asynchronous Programming）的核心关键字。它通常与 `async def` 成对出现。
```python
import asyncio
import time

# 定义一个异步函数（协程）
async def fetch_data(delay, name):
    print(f"任务 {name}: 开始下载数据...")
    # 模拟耗时的网络请求，await 会在这里释放 CPU，让其他任务运行
    await asyncio.sleep(delay) 
    print(f"任务 {name}: 数据下载完成！")
    return f"{name} 的数据"

async def main():
    start_time = time.time()
    
    # 同时并发运行两个下载任务
    # asyncio.gather 会并发调度它们，遇到 await 自动切换
    results = await asyncio.gather(
        fetch_data(2, "A"),
        fetch_data(3, "B")
    )
    
    print(f"所有结果: {results}")
    print(f"总耗时: {time.time() - start_time:.2f} 秒")

# 启动异步事件循环
asyncio.run(main())
```
## 幂运算符

- Python 会**从右往左**计算。
  1. 表达式 `2**3**2`
  2. 等价于`2**(3**2)` -> `2 ** 9` ->512
  3. 而不是从左往右计算`2 ** 3 ** 2` -> `8 ** 2` -> `64`
- 与一元运算符冲突
	```python
	x = 2 ** -3
	# 把 -3 当作整体
	# 0.125
	
	x = -2 ** 3
	# 先算 2 ** 3 = 8
	# -8 
	```

__pow__ 与 __rpow__
这两个魔术方法（Magic Methods）成对出现，共同构成了 Python 的“双目运算符重载机制”**。它们用来定义你的自定义对象在遇到**幂运算符 `**` 或 **`pow()` 函数**时的计算行为。

- **`__pow__(self, other)`**：**左操作数方法（主动方）**。当你的对象位于`**`的**左边**时被触发。
- **`__rpow__(self, other)`**：**右操作数方法（被动方，r 代表 reverse 反向）**。当你的对象位于`**`的**右边**，且左边的普通对象不知道该怎么计算时，由它来救场。

```python
class MagicNumber:  
  
    def __init__(self, value):  
        self.value = value  
  
    def __repr__(self):  
        return f"MagicNumber({self.value})"  
  
    def __pow__(self, other):  
        print("---> 触发了 __pow__")  
        power_val = other.value if isinstance(other, MagicNumber) else other  
        return MagicNumber(self.value ** power_val)  
  
    def __rpow__(self, other):  
        print("--> 触发了 __rpow__")  
        # 此时 self 是指数，other 是底数  
        return MagicNumber(other ** self.value)  
  
m2 = MagicNumber(2)  
m3 = MagicNumber(3)  
  
print(m2 ** m3)  
# ---> 触发了 __pow__# MagicNumber(8)  
  
print(m2 ** 3)  
# ---> 触发了 __pow__# MagicNumber(8)  
  
print(3 ** m2)  
# --> 触发了 __rpow__# MagicNumber(9)  
# 1.Python 先去问左边的内置整型 3：“你会和 MagicNumber 做幂运算吗？  
# 2. ”内置的 int.__pow__ 摇了摇头，返回了 NotImplemented。  
# 3. Python 立刻启动反向救场机制，转头调用右边 m2 的 __rpow__(m2, 3)（注意：此时传入的 other 是底数 3）。计算出 $3^2 = 9$。
```
## 一元运算符

1. `-`对数字取负值
```python
x = 5
print(+x)   # 输出: 5
y = -10
print(+y)   # 输出: -10 (不会变成正数！)
```
2. `+`将参数不加修改的输出其数字参数, 把正数变成负数，把负数变成正数
```python
x = 5
print(-x)   # 输出: -5
y = -10
print(-y)   # 输出: 10
```
3. `~` 取反，将输出对其整数参数按位取反的结果。 对 `x` 按位取反被定义为 `-(x+1)`
```python
print(~5)   # 输出: -6  (计算过程: -(5 + 1))
print(~-1)  # 输出: 0   (计算过程: -(-1 + 1))
```

`__pos__`,`__neg__(self)`,`__invert__(self)`

```python
class Point:  
    def __init__(self, x, y):  
        self.x = x  
        self.y = y  
  
    def __repr__(self):  
        return f"Point({self.x}, {self.y})"  
  
    def __pos__(self):  
        return Point(self.x, self.y)  
  
    def __neg__(self):  
        return Point(-self.x, -self.y)  
  
    def __invert__(self):  
        return Point(~self.x, ~self.y)  
  
p = Point(1, 2)  
print(+p) # Point(1, 2)  
print(-p) # Point(-1, -2)  
print(~p) # Point(-2, -3)
```

## 二元运算符
1. `*`  `__mul__()` 和 `__rmul__()` 
2. `@`  `__matmul__()` 和 `__rmatmul__()` 
3. `//` `__truediv__()` 和 `__rtruediv__()`
4. `/` `__floordiv__()` 和 `__rfloordiv__()` 
5. `%`   `__mod__() `和 `__rmod__()`
6. `+` `__add__()` 和 `__radd__()`
7. `-` `__sub__()` 和 `__rsub__()`

## 增强赋值运算符

增强赋值运算符（Augmented Assignment）是 `=` 与其他运算符的组合简写形式，例如 `+=`、`-=`、`*=` 等。它们既简化了代码，又在底层有特殊的行为差异。

### 基本语法

```python
x = 10
x += 5   # 等价于 x = x + 5
x -= 3   # 等价于 x = x - 3
x *= 2   # 等价于 x = x * 2
x /= 4   # 等价于 x = x / 4
```

### `+=` 对列表的特殊行为

对于列表（list），`+=` **不是**创建一个新列表，而是**原地修改**（in-place），等价于调用 `extend()`：

```python
# 数值类型：+= 创建新对象
a = 10
print(id(a))    # 例如：140732123456789
a += 5
print(id(a))    # 新的 id：140732123456901（不同的对象）

# 列表类型：+= 原地修改
lst = [1, 2]
print(id(lst))  # 例如：140732987654321
lst += [3, 4]   # 等价于 lst.extend([3, 4])
print(id(lst))  # 相同的 id：140732987654321（原对象被修改）
print(lst)      # [1, 2, 3, 4]
```

这与 `+` 运算符不同：

```python
# lst = lst + [3, 4] 会创建新列表
lst = [1, 2]
new_lst = lst + [3, 4]  # 创建新列表，lst 不变
print(lst)      # [1, 2]（未改变）
print(new_lst)  # [1, 2, 3, 4]（新对象）
```

### 魔术方法对照表

| 运算符 | 魔术方法              | 等价于           |
| :----: | :-------------------: | :---------------: |
| `+=`   | `__iadd__`            | `extend()` (list) |
| `-=`   | `__isub__`            |                   |
| `*=`   | `__imul__`            |                   |
| `/=`   | `__itruediv__`        |                   |
| `//=`  | `__ifloordiv__`       |                   |
| `%=`   | `__imod__`            |                   |
| `**=`  | `__ipow__`            |                   |
| `<<=`  | `__ilshift__`         |                   |
| `>>=`  | `__irshift__`         |                   |
| `&=`   | `__iand__`            |                   |
| `\|=`  | `__ior__`             |                   |
| `^=`   | `__ixor__`            |                   |

### 实际应用示例

在多轮对话中追加消息列表：

```python
messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "What is the capital of France?"},
]

response = client.chat.completions.create(model="gpt-4", messages=messages)

# 把模型的回复追加到列表中
messages += [response.choices[0].message]

# 继续追加新的用户输入
messages += [{"role": "user", "content": "And its population?"}]
```

## 位移运算
1. `>>` 右移 把数字 `x` 的二进制形式整体**向右移动 `n` 位**，左边空出来的位用**符号位**（正数补 `0`，负数补 `1`）填补，右边溢出的位直接丢弃。具体来说，`x >> n` 的数学本质是**地板除**, $x//2^n$ 自定义接口`__lshift__`和`__rlshift__`
2. `<<` 左移 **左移 1 位相当于乘以 $2$**。 因此，`x << n` 的数学本质就是：$x * 2^n$,自定义函数`__rshift__`和`__rshift__`

## 二元位运算
1. `&` **全 1 才得 1，否则得 0**（相当于逻辑学中的 `and`）
2. `|` **有 1 就得 1，全 0 才得 0**（相当于逻辑学中的 `or`）。
3. `^` **相同得 0，不同得 1**。

**魔术方法**

|       运算符       |          正向方法          |          反向方法           |             原位赋值变体             |
| :-------------: | :--------------------: | :---------------------: | :----------------------------: |
|       `&`       | `__and__(self, other)` | `__rand__(self, other)` | `__iand__(self, other)`**&=**  |
| <code>\|</code> | `__or__(self, other)`  | `__ror__(self, other)`  | `__ior__(self, other)` **\|=** |
|       `^`       | `__xor__(self, other)` | `__rxor__(self, other)` | `__ixor__(self, other)` **^=** |
## 比较运算符

比较运算会产生布尔值: True 或 False。 自定义的 富比较方法 可能返回非布尔值。 在此情况下 Python 将在布尔运算上下文中对该值调用 bool()。
比较运算可以任意串连，例如 `x < y <= z` 等价于 `x < y and y <= z`，除了 `y` 只被求值一次（但在两种写法下当 `x < y` 值为假时 `z` 都不会被求值）。
正式的说法是这样：如果 _a_, _b_, _c_, ..., _y_, _z_ 为表达式而 _op1_, _op2_, ..., _opN_ 为比较运算符，则 `a op1 b op2 c ... y opN z` 就等价于 `a op1 b and b op2 c and ... y opN z`，不同点在于每个表达式最多只被求值一次。


| 运算符         | 描述                 | 魔术方法                       |
| ----------- | ------------------ | -------------------------- |
| `>`         | 大于                 | `__gt__(self, other)`      |
| `<`         | 小于                 | `__lt__(self, other)`      |
| `>=`        | 大于等于               | `__ge__(self, other)`      |
| `<=`        | 小于等于               | ``__le__(self, other)``    |
| `==`        | 比较对象两边的值是否相等       | `__eq__(self, other)`      |
| `!=`        | 比较两边对象的值是否不等       | `__ne__(self, other)`      |
| `is/is not` | 判断两边对象是否指向同一个内存地址  | **无**                      |
| `in/not in` | 检查左边的元素是否包含在右边的容器中 | `__contains__(self, item)` |
|             |                    |                            |

## 赋值表达式

```EBNF
assignment_expression: [identifier ":="] expression
```

赋值表达式（有时又被称为“命名表达式”或“海象表达式”）将一个 expression 赋值给一个 identifier，同时还会返回 expression 的值。

在传统语法中，**“赋值”是一个语句（Statement）**，而**不是一个表达式（Expression）**。这意味着你不能把赋值写在 `if` 或 `while` 的条件判断里面。
```python
user_info = {"level": 8}

# 必须先独立写一行赋值语句
level = user_info.get("level", 0) 
if level > 5:
    print(f"高级用户欢迎您，您的等级是: {level}")
```

使用 `:=`，我们可以把“获取数据、赋值给变量、进行大小判断”这三件事**压缩到一行代码**中：

```python
user_info = {"level": 8}

# 在 if 判断内部直接完成赋值！
if (level := user_info.get("level", 0)) > 5:
    print(f"高级用户欢迎您，您的等级是: {level}")
```

## 条件表达式

条件表达式（有时称为“三元运算符”）是 if-else 语句的替代物。 由于它是表达式，它会返回一个值并可作为子表达式出现。
表达式 `x if C else y` 首先是对条件 _C_ 而非 _x_ 求值。 如果 _C_ 为真，_x_ 将被求值并返回其值；否则将对 _y_ 求值并返回其值。

```EBNF
conditional_expression: or_test ["if" or_test "else" expression]
expression:             conditional_expression | lambda_expr
```

## lambda表达式

```EBNF
lambda_expr: "lambda" [parameter_list] ":" expression
```
lambda 表达式（有时称为 lambda 构型）被用于创建匿名函数。 表达式 `lambda parameters: expression` 会产生一个函数对象 。 

传统写法
```python
def add(x, y):
    return x + y

print(add(5, 3)) # 输出: 8
```

Lambda写法
```python
# 把 lambda 赋值给变量（虽然实际开发不推荐这样用，但有助于理解）
add_lambda = lambda x, y: x + y

print(add_lambda(5, 3)) # 输出: 8
```