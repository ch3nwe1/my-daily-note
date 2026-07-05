
## datetime
```python
from datetime import datetime  
  
now = datetime.now()  
print(now) # 2026-05-28 17:42:25.514737  
print(f'now.year: {now.year}') # now.year: 2026  
print(f'now.month: {now.month}') # now.month: 5  
print(f'now.day: {now.day}') # now.day: 28  
print(f'now.hour: {now.hour}') # now.day: 28  
print(f'now.minute: {now.minute}') # now.minute: 42  
print(f'now.second: {now.second}') # now.second: 25

dt = datetime(2026, 10, 1, 8, 30)  
print(dt)           # 2026-10-01 08:30:00
```

## date and time

```python
from datetime import date,time  
# import time  
  
today = date.today()  
print(today) #2026-05-28  
  
# 纯时间  
t = time(14, 5, 20)  
print(t) # 14:05:20
```

## timedelta
```python
from datetime import datetime, timedelta

now = datetime.now()

# 1. 计算 5 天后的时间
future = now + timedelta(days=5)
print(f"5天后: {future}")

# 2. 计算 2 小时 30 分钟前的时间
past = now - timedelta(hours=2, minutes=30)
print(f"2.5小时前: {past}")

# 3. 计算两个日期的差距
d1 = datetime(2026, 6, 1)
d2 = datetime(2026, 5, 28)
diff = d1 - d2
print(type(diff))   # <class 'datetime.timedelta'>
print(diff.days)    # 输出: 4 (相差4天)
```

## 时间与字符串的转换

- `strftime`: **Str**ing **F**ormat **Time**（时间 ➡️ 字符串，**F** 代表 Format 格式化）
- `strptime`: **Str**ing **P**arse **Time**（字符串 ➡️ 时间，**P** 代表 Parse 解析）

**格式化占位符**
- `%Y`:四位数的年份
- `%m`:两位数的月份
- `%d`: 两位数的日期
- `%H`:24小时制的小时
- `%M`: 分钟
- `%S`: 秒
```python
from datetime import datetime

now = datetime.now()

# 【1】strftime: 时间对象 -> 漂亮的字符串
str_time = now.strftime("%Y-%m-%d %H:%M:%S")
print(str_time)  # 输出: "2026-05-28 17:19:07"

# 【2】strptime: 字符串 -> 时间对象（必须保证格式字符串与文本完全一一对应）
input_str = "2026/12/25 20:00"
dt_object = datetime.strptime(input_str, "%Y/%m/%d %H:%M")
print(dt_object) # 输出: 2026-12-25 20:00:00
print(type(dt_object)) # <class 'datetime.datetime'>
```