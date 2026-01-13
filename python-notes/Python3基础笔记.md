---
title: Python3 基础笔记
tags:
  - python
  - programming
  - basics
category: Python
created: 2026-01-13
---

# Python3 基础笔记

## 简介

Python 是一种解释型、面向对象、动态数据类型的高级程序设计语言。由 Guido van Rossum 于 1989 年发明。

### 特点
- 语法简洁优雅
- 跨平台兼容性强
- 丰富的标准库和第三方库
- 支持多种编程范式

### 第一个程序

```python
print("Hello, World!")
```

---

## 基础语法

### 注释

```python
# 单行注释

"""
多行注释
或文档字符串
"""
```

### 变量与数据类型

Python 是动态类型语言，变量无需声明类型。

```python
# 数值类型
age = 25           # int
price = 19.99      # float
complex_num = 3+4j # complex

# 字符串
name = "Alice"
message = 'Hello, World!'

# 布尔值
is_active = True
is_deleted = False

# None 空值
result = None
```

### 类型转换

```python
int("123")    # 字符串转整数 -> 123
float("3.14") # 字符串转浮点数 -> 3.14
str(100)      # 整数转字符串 -> "100"
bool(1)       # 布尔值 -> True
```

---

## 运算符

### 算术运算符

```python
a, b = 10, 3
print(a + b)  # 加法 -> 13
print(a - b)  # 减法 -> 7
print(a * b)  # 乘法 -> 30
print(a / b)  # 除法 -> 3.333...
print(a // b) # 整除 -> 3
print(a % b)  # 取余 -> 1
print(a ** b) # 幂运算 -> 1000
```

### 比较运算符

```python
print(a == b) # 等于 -> False
print(a != b) # 不等于 -> True
print(a > b)  # 大于 -> True
print(a < b)  # 小于 -> False
print(a >= b) # 大于等于 -> True
print(a <= b) # 小于等于 -> False
```

### 逻辑运算符

```python
x, y = True, False
print(x and y) # 逻辑与 -> False
print(x or y)  # 逻辑或 -> True
print(not x)   # 逻辑非 -> False
```

### 赋值运算符

```python
a = 10
a += 5        # a = a + 5 -> 15
a -= 3        # a = a - 3 -> 12
a *= 2        # a = a * 2 -> 24
```

---

## 字符串

### 字符串操作

```python
s = "Hello, Python!"

# 切片
s[0]          # 'H'
s[0:5]        # 'Hello'
s[:5]         # 'Hello'
s[7:]         # 'Python!'
s[::-1]       # 反转 -> '!nohtyP ,olleH'

# 常用方法
len(s)              # 长度 -> 13
s.lower()           # 小写
s.upper()           # 大写
s.strip()           # 去除两端空白
s.replace("H", "J") # 替换
s.split(",")        # 分割
s.join(["a", "b"])  # 拼接 -> "a,b"
"py" in s           # 包含判断 -> False
s.format(name="Python") # 格式化
```

### 字符串格式化

```python
# f-string (推荐)
name = "Alice"
age = 25
print(f"My name is {name}, I'm {age}")

# format 方法
print("My name is {}, I'm {}".format(name, age))

# % 格式化
print("My name is %s, I'm %d" % (name, age))
```

---

## 列表 List

```python
fruits = ["apple", "banana", "orange"]

# 访问
fruits[0]       # 'apple'
fruits[-1]      # 'orange' (倒数)

# 修改
fruits[1] = "grape"

# 添加
fruits.append("mango")       # 末尾添加
fruits.insert(1, "pear")     # 指定位置插入
fruits.extend(["kiwi"])      # 扩展列表

# 删除
fruits.remove("banana")      # 移除第一个匹配项
fruits.pop()                 # 弹出最后一个
del fruits[0]                # 删除指定位置
fruits.clear()               # 清空列表

# 其他操作
len(fruits)       # 长度
fruits.sort()     # 排序
fruits.reverse()  # 反转
fruits.index("apple")  # 查找索引
"apple" in fruits      # 是否存在
```

---

## 元组 Tuple

```python
# 元组是不可变的列表
coordinates = (10, 20, 30)
person = ("Alice", 25, "Engineer")

# 访问
coordinates[0]  # 10

# 解包
x, y, z = coordinates  # x=10, y=20, z=30
```

---

## 字典 Dictionary

```python
person = {
    "name": "Alice",
    "age": 25,
    "city": "Beijing"
}

# 访问
person["name"]        # "Alice"
person.get("email")   # None (不报错)
person.get("email", "N/A") # 默认值

# 修改/添加
person["age"] = 26
person["email"] = "alice@example.com"

# 删除
del person["city"]
age = person.pop("age")

# 常用方法
person.keys()     # 所有键
person.values()   # 所有值
person.items()    # 所有键值对
"name" in person  # 键是否存在
```

---

## 集合 Set

```python
# 集合是无序、不重复的元素
colors = {"red", "blue", "green"}

# 添加
colors.add("yellow")
colors.update(["pink", "purple"])

# 删除
colors.remove("red")      # 不存在会报错
colors.discard("red")     # 不存在不报错
colors.pop()              # 随机删除

# 集合运算
a = {1, 2, 3}
b = {2, 3, 4}
a | b  # 并集 -> {1, 2, 3, 4}
a & b  # 交集 -> {2, 3}
a - b  # 差集 -> {1}
a ^ b  # 对称差集 -> {1, 4}
```

---

## 流程控制

### 条件语句

```python
score = 85

if score >= 90:
    print("优秀")
elif score >= 80:
    print("良好")
elif score >= 60:
    print("及格")
else:
    print("不及格")
```

### 三元表达式

```python
status = "成年" if age >= 18 else "未成年"
```

### 循环

```python
# for 循环
for i in range(5):
    print(i)  # 0, 1, 2, 3, 4

for fruit in ["apple", "banana"]:
    print(fruit)

# enumerate 获取索引
for i, fruit in enumerate(["a", "b"]):
    print(i, fruit)

# while 循环
count = 0
while count < 3:
    print(count)
    count += 1

# break 和 continue
for i in range(10):
    if i == 5:
        break      # 退出循环
    if i % 2 == 0:
        continue   # 跳过本次
    print(i)
```

### 列表/字典/集合推导式

```python
# 列表推导式
squares = [x**2 for x in range(5)]
even = [x for x in range(10) if x % 2 == 0]

# 字典推导式
squares_dict = {x: x**2 for x in range(3)}

# 集合推导式
unique = {x for x in [1, 2, 2, 3]}
```

---

## 函数

### 定义函数

```python
def greet(name):
    """打招呼函数"""
    return f"Hello, {name}!"

# 调用
print(greet("Alice"))
```

### 参数类型

```python
# 默认参数
def func(a, b=10):
    return a + b

# 关键字参数
func(a=5, b=3)

# 不定长参数
def func(*args, **kwargs):
    print(args)    # 元组
    print(kwargs)  # 字典

# 位置实参解包
nums = [1, 2, 3]
func(*nums)

# 关键字实参解包
info = {"name": "Alice", "age": 25}
func(**info)
```

### 作用域

```python
x = "全局变量"

def func():
    x = "局部变量"
    print(x)  # 局部变量

def func_global():
    global x
    x = "修改全局"
```

### 匿名函数

```python
add = lambda a, b: a + b
print(add(3, 5))  # 8

# 与 map/filter 配合
nums = [1, 2, 3, 4]
squared = list(map(lambda x: x**2, nums))
evens = list(filter(lambda x: x % 2 == 0, nums))
```

---

## 模块与包

### 导入模块

```python
import math
print(math.sqrt(16))

from math import sqrt, pi
print(sqrt(16))
print(pi)

import math as m
print(m.sqrt(16))

from math import *  # 不推荐
```

### 创建模块

```python
# mymodule.py
def hello():
    print("Hello!")

# 导入
import mymodule
mymodule.hello()
```

---

## 文件操作

### 读写文件

```python
# 读取
with open("file.txt", "r", encoding="utf-8") as f:
    content = f.read()           # 读取全部
    lines = f.readlines()        # 读取所有行

# 写入
with open("file.txt", "w", encoding="utf-8") as f:
    f.write("Hello\n")
    f.writelines(["line1\n", "line2\n"])

# 追加
with open("file.txt", "a", encoding="utf-8") as f:
    f.write("追加内容\n")

# 读写模式
# r: 读, w: 写(覆盖), a: 追加, x: 创建新文件
```

### JSON 文件

```python
import json

# 写入
data = {"name": "Alice", "age": 25}
with open("data.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)

# 读取
with open("data.json", "r", encoding="utf-8") as f:
    data = json.load(f)
```

---

## 异常处理

```python
try:
    result = 10 / 0
except ZeroDivisionError:
    print("除以零错误")
except Exception as e:
    print(f"其他错误: {e}")
else:
    print("没有异常时执行")
finally:
    print("无论是否异常都执行")
```

### 抛出异常

```python
def check_age(age):
    if age < 0:
        raise ValueError("年龄不能为负数")
    return age

# 自定义异常
class MyError(Exception):
    pass
```

---

## 面向对象编程

### 类与对象

```python
class Person:
    # 类属性
    species = "Human"

    def __init__(self, name, age):
        # 实例属性
        self.name = name
        self.__age = age  # 私有属性

    def greet(self):
        return f"I'm {self.name}"

    # 装饰器定义属性
    @property
    def age(self):
        return self.__age

    @age.setter
    def age(self, value):
        if value > 0:
            self.__age = value

# 创建对象
person = Person("Alice", 25)
print(person.greet())
print(person.age)
```

### 继承

```python
class Student(Person):
    def __init__(self, name, age, school):
        super().__init__(name, age)
        self.school = school

    def greet(self):
        return f"I'm {self.name}, a student at {self.school}"
```

### 特殊方法

```python
class MyClass:
    def __str__(self):        # str() 或 print() 调用
        return "MyClass object"

    def __repr__(self):       # 官方字符串表示
        return "MyClass()"

    def __len__(self):
        return 10

    def __getitem__(self, index):
        return index * 2
```

---

## 常用标准库

```python
import os        # 操作系统功能
import sys       # 系统参数
import datetime  # 日期时间
import json      # JSON 处理
import re        # 正则表达式
import math      # 数学函数
import random    # 随机数
import collections # 容器数据类型
import itertools # 迭代工具
import functools # 高阶函数
import shutil    # 文件操作
import hashlib   # 哈希算法
```

---

## 总结

Python3 基础包括：
1. **语法基础** - 变量、数据类型、运算符
2. **数据结构** - 列表、元组、字典、集合
3. **流程控制** - 条件、循环
4. **函数** - 定义、参数、作用域
5. **模块** - 导入、使用
6. **文件操作** - 读写、JSON
7. **异常处理** - try/except
8. **面向对象** - 类、继承

> 持续学习，多写代码练习！
