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

