> 单个字符匹配
- `.`:匹配除了换行符以外的单个字符
- `\d`:匹配一个数字,相当于`[0-9]`
- `\w`: 匹配一个字母数字或下划线相当于`[a-zA-Z0-9_]`
- `\s`: 匹配一个空白字符，如空格，制表符`\t`，换行符`\n`
- `[abc]`: 字符集，匹配a,b,c中的任意一个
> 数量限定符
- `*`:重复0次或多次
- `+` 重复一次或多次（至少一次）
- `?` 重复0次或1次
- `{n}`:精确重复n次
- `{n,m}`重复n到m次
> 位置边界锚定
- `^`匹配字符串的开头
- `$`匹配字符串的结尾

## re.search

```python
  
import re  
  
text = "我的电话是: 13812345678，他的电话是: 13987654321"  
  
result = re.search(r'\d{11}', text, )  
if result:  
    print(result.group()) # 13812345678
```

它会从左到右扫描整个字符串，一旦找到**第一个**匹配的内容就会立即停止，并返回一个 `Match` 对象。如果全文本都没找到，返回 `None`

## re.findall()
如果你想把文本里所有符合条件的内容都榨取出来，用 `findall` 最合适。它会直接返回一个**包含所有匹配字串的列表**。

```python
import re  
  
text = "Apple costs $2, Orange costs $5, Banana costs $12."  
  
# \d+ 匹配一个或多个数字  
prices = re.findall(r"\d+", text)  
print(prices)  # 输出: ['2', '5', '12']
```

# re.match

`re.match()` 和 `re.search()` 很像，但有一个极度严苛的区别：**`match` 必须从字符串的第一个字符开始匹配**。如果开头不符合，后面再符合也拿不到。
```python
import re  
  
text = "Hello 12345"  
  
print(re.match(r"\d+", text))   # 输出: None (因为开头是 "Hello" 不是数字)  
print(re.search(r"\d+", text))  # 输出:<re.Match object; span=(6, 11), match='12345'> (能搜到 12345)
```

## re.sub

类似于字串的 `.replace()`，但支持用正则表达式去模糊匹配并替换。
```python
import re  
  
text = "用户 A 的密码是 123456，用户 B 的密码是 abcd789"  
# 把所有的数字都替换成六个星号  
safe_text = re.sub(r"\d+", "******", text)  
print(safe_text)  # 输出: 用户 A 的密码是 ******，用户 B 的密码是 abcd******
```

## group

捕获组是正则的分水岭。当你用圆括号 `()` 包裹正则表达式的一部分时，你不仅是在限定它们的整体结构，更是标记了一个“提取子窗口”。
```python
import re  
  
email = "谷歌的支持邮箱是:support@google.com"  
  
# 用两个括号，分别提取用户名和域名  
pattern = r"(\w+)@(\w+\.\w+)"  
  
match = re.search(pattern, email)  
if match:  
    print(f"完整邮箱: {match.group(0)}") # group(0) 代表匹配到的完整文本 support@google.com    print(f"用 户 名: {match.group(1)}") # group(1) 拿到第一个括号的内容: support  
    print(f"域    名: {match.group(2)}") # group(2) 拿到第二个括号的内容: google.com
```