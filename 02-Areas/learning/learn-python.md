---
title: learn-python
tags:
  - python
  - learning
aliases: []
cssclass: []
created: "2026-06-10"
---

# learn-python

## 关键词和保留字

```python
import keyword

print(keyword.kwlist)  # 输出所有关键字
print(len(keyword.kwlist))  # 关键字数量
```

## 标识符

标识符用来给变量、函数、类、模块等命名。

规则：
- 由字母、数字、下划线组成
- 不能以数字开头
- 区分大小写
- 不能使用关键字

常见命名规范：
- 变量名、函数名：小写字母+下划线（snake_case）
- 类名：大驼峰（CamelCase）
- 常量：全大写+下划线

## 注释

```python
# 单行注释

"""
多行注释（文档字符串）
"""

'''
多行注释
'''
```

## 变量

变量是存储数据的容器，Python 中变量不需要声明类型。

```python
name = "Alice"   # 字符串
age = 25         # 整数
height = 1.75    # 浮点数
is_student = True  # 布尔值

# 多变量赋值
a, b, c = 1, 2, 3
x = y = z = 0
```

## 数据类型

| 类型 | 示例 | 说明 |
|------|------|------|
| int | `42` | 整数 |
| float | `3.14` | 浮点数 |
| str | `"hello"` | 字符串 |
| bool | `True` / `False` | 布尔值 |
| list | `[1, 2, 3]` | 列表 |
| tuple | `(1, 2, 3)` | 元组 |
| dict | `{"a": 1}` | 字典 |
| set | `{1, 2, 3}` | 集合 |
| NoneType | `None` | 空值 |

### 类型转换

```python
int("42")       # 字符串 -> 整数
float("3.14")   # 字符串 -> 浮点数
str(100)        # 整数 -> 字符串
bool(0)         # 0 -> False, 非0 -> True
list("abc")     # ['a', 'b', 'c']
```

## 运算符

### 算术运算符

| 运算符 | 说明 | 示例 |
|--------|------|------|
| `+` | 加 | `3 + 2 = 5` |
| `-` | 减 | `3 - 2 = 1` |
| `*` | 乘 | `3 * 2 = 6` |
| `/` | 除 | `3 / 2 = 1.5` |
| `//` | 整除 | `3 // 2 = 1` |
| `%` | 取余 | `3 % 2 = 1` |
| `**` | 幂 | `3 ** 2 = 9` |

### 比较运算符

`==` `!=` `>` `<` `>=` `<=`

### 逻辑运算符

`and` `or` `not`

### 赋值运算符

`=` `+=` `-=` `*=` `/=` `//=` `%=` `**=`

## 字符串操作

```python
s = "Hello, World!"

# 索引和切片
s[0]       # 'H'
s[-1]      # '!'
s[0:5]     # 'Hello'
s[7:]      # 'World!'

# 常用方法
s.lower()         # 转小写
s.upper()         # 转大写
s.strip()         # 去除首尾空格
s.split(", ")     # 分割字符串
s.replace("H", "J")  # 替换
s.find("World")   # 查找子串位置
len(s)            # 字符串长度

# 格式化
name = "Alice"
f"Hello, {name}"           # f-string
"Hello, {}".format(name)   # format()
"Hello, %s" % name         # %
```

## 数据结构

### 列表 list

```python
fruits = ["apple", "banana", "cherry"]

fruits.append("date")     # 末尾添加
fruits.insert(0, "fig")   # 指定位置插入
fruits.remove("banana")   # 删除元素
fruits.pop()              # 弹出末尾元素
fruits.sort()             # 排序
fruits.reverse()          # 反转
len(fruits)               # 长度

# 列表推导式
squares = [x**2 for x in range(10)]
```

### 元组 tuple

```python
point = (3, 4)      # 不可变
x, y = point        # 解包
```

### 字典 dict

```python
person = {"name": "Alice", "age": 25}

person["name"]             # 获取值
person["city"] = "Beijing" # 添加键值对
del person["age"]          # 删除键值对
person.keys()              # 所有键
person.values()            # 所有值
person.items()             # 所有键值对

# 字典推导式
squares = {x: x**2 for x in range(5)}
```

### 集合 set

```python
s = {1, 2, 3}
s.add(4)           # 添加元素
s.remove(1)        # 删除元素

# 集合运算
a = {1, 2, 3}
b = {3, 4, 5}
a | b   # 并集 {1, 2, 3, 4, 5}
a & b   # 交集 {3}
a - b   # 差集 {1, 2}
```

## 流程控制

### 条件语句

```python
score = 85

if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
elif score >= 70:
    grade = "C"
else:
    grade = "D"
```

### 循环语句

```python
# for 循环
for i in range(5):
    print(i)

for item in ["a", "b", "c"]:
    print(item)

# while 循环
count = 0
while count < 5:
    print(count)
    count += 1

# break 和 continue
for i in range(10):
    if i == 3:
        continue  # 跳过本次
    if i == 7:
        break     # 终止循环
    print(i)
```

## 函数

```python
# 基本函数
def greet(name):
    return f"Hello, {name}!"

# 默认参数
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}!"

# 可变参数
def add(*args):
    return sum(args)

# 关键字参数
def info(**kwargs):
    for key, value in kwargs.items():
        print(f"{key}: {value}")

# Lambda 函数
square = lambda x: x ** 2
```

## 类和对象

```python
class Dog:
    # 类变量
    species = "Canis familiaris"

    # 初始化方法
    def __init__(self, name, age):
        self.name = name    # 实例变量
        self.age = age

    # 实例方法
    def speak(self):
        return f"{self.name} says Woof!"

    # 类方法
    @classmethod
    def from_birth_year(cls, name, birth_year):
        age = 2026 - birth_year
        return cls(name, age)

    # 静态方法
    @staticmethod
    def is_adult(age):
        return age >= 3

# 创建对象
dog = Dog("Buddy", 5)
print(dog.speak())

# 继承
class Puppy(Dog):
    def __init__(self, name, age, training_level):
        super().__init__(name, age)
        self.training_level = training_level
```

## 异常处理

```python
try:
    result = 10 / 0
except ZeroDivisionError:
    print("不能除以零")
except Exception as e:
    print(f"发生错误: {e}")
else:
    print("没有异常")
finally:
    print("总会执行")

# 抛出异常
raise ValueError("无效的值")
```

## 文件操作

```python
# 写入文件
with open("file.txt", "w") as f:
    f.write("Hello, World!")

# 读取文件
with open("file.txt", "r") as f:
    content = f.read()
    # 或逐行读取
    for line in f:
        print(line.strip())
```

## 常用内置函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `print()` | 输出 | `print("hello")` |
| `input()` | 输入 | `name = input()` |
| `len()` | 长度 | `len([1,2,3])` |
| `type()` | 类型 | `type(42)` |
| `range()` | 范围 | `range(0, 10)` |
| `enumerate()` | 枚举 | `enumerate(["a","b"])` |
| `zip()` | 打包 | `zip([1,2], [3,4])` |
| `map()` | 映射 | `map(str, [1,2,3])` |
| `filter()` | 过滤 | `filter(bool, [0,1,2])` |
| `sorted()` | 排序 | `sorted([3,1,2])` |
| `reversed()` | 反转 | `reversed([1,2,3])` |
| `min()` / `max()` | 最小/最大值 | `min(1,2,3)` |
| `sum()` | 求和 | `sum([1,2,3])` |
| `abs()` | 绝对值 | `abs(-5)` |
| `isinstance()` | 类型判断 | `isinstance(42, int)` |

## 模块和包

```python
# 导入模块
import math
from math import sqrt, pi
from math import *  # 不推荐

# 常用标准库
import os           # 操作系统接口
import sys          # 系统相关
import datetime     # 日期时间
import json         # JSON 处理
import re           # 正则表达式
import random       # 随机数
import collections  # 集合扩展
import itertools    # 迭代器工具
import functools    # 函数工具

# 创建自己的模块
# my_module.py
def my_function():
    return "Hello from my module"
```

## 列表推导式和生成器

```python
# 列表推导式
squares = [x**2 for x in range(10)]
evens = [x for x in range(20) if x % 2 == 0]

# 字典推导式
square_dict = {x: x**2 for x in range(5)}

# 集合推导式
square_set = {x**2 for x in range(-5, 5)}

# 生成器表达式
gen = (x**2 for x in range(10))
print(next(gen))  # 0
print(next(gen))  # 1

# 生成器函数
def fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b
```

## 装饰器

```python
import time

def timer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} 运行时间: {end - start:.4f}秒")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
    return "完成"

# 使用
result = slow_function()
```

## 常用技巧

```python
# 交换变量
a, b = b, a

# 合并字典
merged = {**dict1, **dict2}  # Python 3.5+
merged = dict1 | dict2        # Python 3.9+

# 海象运算符 := (Python 3.8+)
if (n := len(data)) > 10:
    print(f"数据量 {n} 太大")

# 解包
first, *rest = [1, 2, 3, 4, 5]
# first = 1, rest = [2, 3, 4, 5]

# 链式比较
if 0 < x < 10:
    print("x 在 0 到 10 之间")
```
