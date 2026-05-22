# 格式化字符串语法
格式字符串包含有以花括号 `{}` 括起来的“替换字段”。不在花括号之内的内容被视为字面文本，会不加修改地复制到输出中。 如果你需要在字面文本中包含花括号字符，可以通过重复来转义: `{{` 和 `}}`。
```txt
replacement_field: "{" [field_name] ["!" conversion] [":" format_spec] "}"
field_name:        arg_name ("." attribute_name | "[" element_index "]")*
arg_name:          [identifier | digit+]
attribute_name:    identifier
element_index:     digit+ | index_string
index_string:      <any source character except "]"> +
conversion:        "r" | "s" | "a"
format_spec:       format-spec:format_spec
```
_field_name_ 本身以一个数字或关键字形式的 _arg_name_ 打头。 如果为数字，则它指向一个位置参数，而如果为关键字，则它指向一个命名关键字参数。
```python
"First, thou shalt count to {0}"
"My quest is {name}"
```
如果格式字段串中的数字 arg_names 为 0, 1, 2, ... 的序列，它们可以全部（而非部分）被省略并且数字 0, 1, 2, ... 将按顺序被自动插入
```python
"Bring me a {}"                   # 隐式引用第一个位置参数
"From {} to {}" # 等同于 "From {0} to {1}"
```
 _arg_name_ 之后可以跟任意数量的索引或属性表达式。
 ```python
"Weight in tons {0.weight}"       # 第一个位置参数的 'weight' 属性
"Units destroyed: {players[0]}"   # 关键字参数 'players' 的第一个元素。
 ```

conversion 字段会在格式化之前进行类型强制转换。通常，格式化一个值的工作是由该值本身的 __format__() 方法完成的。但是，在某些情况下最好是强制将类型格式化为一个字符串，覆盖其本身的格式化定义。 通过在调用 __format__() 之前将值转换为字符串，可以绕过正常的格式化逻辑。
```python
"Harold's a clever {0!s}"        # 先在参数上调用 str()
"Bring out the holy {name!r}"    # 先在参数上调用 repr()
"More {!a}"                      # 先在参数上调用 ascii()
```

_format_spec_ 字段还可以在其内部包含嵌套的替换字段。 这些嵌套的替换字段可能包括字段名称、转换旗标和格式规格描述，但是不再允许更深层的嵌套。format_spec 内部的替换字段会在解读 _format_spec_ 字符串之前先被解读。这将允许动态地指定特定值的格式。

```txt
format_spec:             [options][width_and_precision][type]
options:                 [[fill]align][sign]["z"]["#"]["0"]
fill:                    <any character>
align:                   "<" | ">" | "=" | "^"
sign:                    "+" | "-" | " "
width_and_precision:     [width_with_grouping][precision_with_grouping]
width_with_grouping:     [width][grouping]
precision_with_grouping: "." [precision][grouping] | "." grouping
width:                   digit+
precision:               digit+
grouping:                "," | "_"
type:                    "b" | "c" | "d" | "e" | "E" | "f" | "F" | "g"
                         | "G" | "n" | "o" | "s" | "x" | "X" | "%"
```

```txt
[[fill]align][sign][\#][0][width][grouping\_option][.precision][type]
```

1. 填充（fill）与对齐（align）
> 这两个通常组合使用，用来把文本“撑”到指定的宽度。**注意：如果你指定了填充字符，就必须紧跟一个对齐符号。**
- < : 左对齐
- >: 右对齐
- ^: 居中对齐
- =：强迫将填充内容放置在符号（如果有的话）与数字之间（常用于财务报表）。
```python
#10代表宽度 fill没写默认空格 assgin
print("{:<10}".format("左")) # '左 ' 
print("{:>10}".format("右")) # ' 右' 
print("{:^10}".format("中")) # ' 中 '
# fill = *,assign = ^ width = 10
print("{:*^10}".format("Hello")) # '**Hello***' 
print("{:_>10}".format(42)) # '________42'

# fill省略 assign是=，sign是+ 宽度是10
print("{:=+10}".format(42)) # '+_______42' （正号在最左，数字在最右）
```
2. 正负号（sign）
> 仅对**数字**有效，用来控制正负号的显示方式：(非数字报错)
- +：正数前加 +，负数前加 -。
- -：只有负数前加 -（这是 Python 的**默认**行为）。
- （空格）：正数前加一个**空格**（为了和负数的减号对齐占位），负数前加 -。
```python
print("{:+} | {:+}".format(520, -520)) # '+520 | -520'
print("{:-} | {:-}".format(520, -520)) # '520 | -520'
print("{: } | {: }".format(520, -520)) # ' 520 | -520' (注意520前面有个隐藏空格)
```
3. 备用形式（#）
> 对于**整数**（二进制 b、八进制 o、十六进制 x/X），它会自动加上前缀 0b、0o、0x 或 0X。
> 对于**浮点数**，即使没有小数位，它也强制保留小数点。
```python
print("{:#b}".format(10)) # '0b1010' 
print("{:#x}".format(255)) # '0xff' 
print("{:#}".format(42.0)) # 强制保留小数点 42.0
```
4. 宽度（width）与 0 填充
> **width**：一个十进制数字，定义当前字段的**最小总宽度**。如果内容比宽度短，就用填充字符补齐；如果内容比宽度长，**不会被截断**，而是会完整显示（撑破边界）。
   **0**：在宽度前面直接加个 0，是 0= 的快捷方式，专用于数字的**前导零**填充。
```python
print("{:10}".format("cat")) # 'cat ' (宽度为10) 
print("{:010}".format(42)) # '0000000042' (总宽度10，前面补0)
```
5. 千位分隔符（grouping_option）
> ,（逗号）：使用逗号作为千位分隔符。
> _（下划线）：使用下划线作为千位分隔符（适用于科学计算或代码可读性）
```python
print("{:,}".format(100000000)) # '100,000,000' 
print("{:_}".format(100000000)) # '100_000_000'
```
6. 精度（.precision）
> 对于**浮点数**（如 f, e）：它代表**小数点后面保留几位**（会进行四舍五入）。
> 对于**字符串**：它代表**最大截断长度**（超过这个长度的字符会被直接扔掉）。
```python
# 浮点数保留两位小数 
print("{:.2f}".format(3.1415926)) # '3.14' 
# 字符串截取前 5 个字符 
print("{:.5}".format("Unbelievable")) # 'Unbel'
```
7. 类型（type）
> d：十进制整数（默认）。
> b：二进制。
> o：八进制。
> x / X：十六进制（小写 / 大写）。
> c：把数字转换为对应的 Unicode 字符。
```python
print("{:d}".format(42)) # '42' 
print("{:b}".format(42)) # '101010' 
print("{:x}".format(42)) # '2a' 
print("{:c}".format(97)) # 'a' (ASCII 码)
```
> f / F：定点表示法（最常用的带小数点的显示，默认保留 6 位）。
> e / E：科学计数法
> g / G：通用格式。自动根据数字大小决定用 f 还是 e。
> %：**百分比格式**。自动将数字乘以 100，并转成 f 格式，最后加上 % 号。
```python
print("{:f}".format(3.14)) # '3.140000' 
print("{:.2e}".format(3140)) # '3.14e+03' 
print("{:.2%}".format(0.1234)) # '12.34%'
```