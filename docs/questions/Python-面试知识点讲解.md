---
title: Python 面试知识点讲解
url: wikibar://summary/questions/Python-面试知识点讲解
source_type: summary
folder: questions
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T15:31:25.387997+00:00'
---

# Python 面试知识点讲解

> 适用方向：AI Agent / LLM / Python Backend

---

# 一、Python 常用数据结构

## 1. List、Tuple、Set、Dict 有什么区别？

Python 最常见的四种容器：

| 类型 | 是否有序 | 是否可变 | 是否允许重复 | 典型用途 |
|---|---|---|---|---|
| list | 是 | 是 | 是 | 保存一组数据 |
| tuple | 是 | 否 | 是 | 保存不可修改的数据 |
| set | 无位置索引 | 是 | 否 | 去重、快速查找 |
| dict | 保持插入顺序 | 是 | Key 不重复 | Key-Value 映射 |

### List

```python
nums = [1, 2, 3]

nums.append(4)

print(nums)
# [1, 2, 3, 4]
```

List 可以修改：

```python
nums[0] = 100
```

---

### Tuple

```python
point = (10, 20)
```

不能：

```python
point[0] = 100
```

会报错，因为 Tuple 是 Immutable。

---

### Set

```python
nums = {1, 2, 3}
```

最大的特点：

```text
元素不重复
+
平均 O(1) 查找
```

例如：

```python
nums = [1, 2, 2, 3, 3]

result = set(nums)

print(result)
# {1, 2, 3}
```

所以经常用于：

```text
去重
成员判断
集合运算
```

---

### Dict

```python
user = {
    "name": "Alice",
    "age": 25
}
```

访问：

```python
user["name"]
```

结果：

```text
Alice
```

Dict 本质是：

```text
Key → Value
```

映射。

---

# 二、Dict 为什么查找通常是 O(1)？

因为 Dict 底层主要基于：

```text
Hash Table
```

假设：

```python
user = {
    "name": "Alice",
    "age": 25
}
```

查询：

```python
user["name"]
```

不是从头遍历：

```text
name?
age?
...
```

而是先：

```text
"name"
 ↓
hash("name")
 ↓
得到一个 Hash Value
 ↓
定位 Hash Table 中的位置
 ↓
找到 Value
```

所以平均时间复杂度：

```text
O(1)
```

但是极端 Hash Collision 等情况下可能退化。

### 面试回答

> Python 的 dict 主要基于哈希表实现，通过对 Key 计算 Hash 来定位存储位置，所以查找、插入和删除平均时间复杂度通常是 O(1)。

---

# 三、为什么 Dict 的 Key 必须是 Hashable？

例如：

```python
d = {
    "name": "Alice"
}
```

String 可以作为 Key。

Tuple 通常也可以：

```python
d = {
    (1, 2): "point"
}
```

但是 List 不可以：

```python
d = {
    [1, 2]: "point"
}
```

会报错。

原因：

```text
Dict
依赖 Key 的 Hash Value
```

如果 Key 可以随便改变：

```text
原来的 Hash
可能失效
```

因此 Dict Key 必须是 Hashable。

常见：

```text
str      ✓
int      ✓
float    ✓
tuple    通常 ✓

list     ✗
dict     ✗
set      ✗
```

---

# 四、List 和 Set 查找有什么区别？

例如：

```python
nums = [1, 2, 3, 4, 5]
```

执行：

```python
5 in nums
```

List 可能需要：

```text
1
↓
2
↓
3
↓
4
↓
5
```

所以：

```text
O(n)
```

Set：

```python
nums = {1, 2, 3, 4, 5}
```

执行：

```python
5 in nums
```

利用 Hash Table：

```text
平均 O(1)
```

所以如果大量进行：

```python
if x in ...
```

通常 Set 更适合。

---

# 五、Mutable 和 Immutable

## Mutable

创建之后可以修改：

```text
list
dict
set
```

例如：

```python
a = [1, 2, 3]

a.append(4)
```

原对象发生变化。

---

## Immutable

创建之后不能修改：

```text
int
float
str
tuple
bool
```

例如：

```python
a = "hello"
```

不能：

```python
a[0] = "H"
```

---

# 六、Python 变量到底保存什么？

看：

```python
a = [1, 2, 3]

b = a
```

可以理解成：

```text
a ─────┐
       ↓
   [1, 2, 3]
       ↑
b ─────┘
```

`a` 和 `b` 指向同一个对象。

所以：

```python
b.append(4)
```

再：

```python
print(a)
```

得到：

```python
[1, 2, 3, 4]
```

因为修改的是同一个 List。

---

# 七、== 和 is 有什么区别？

这是非常经典的 Python 面试题。

## ==

比较：

```text
值是否相同
```

## is

比较：

```text
是不是同一个对象
```

例如：

```python
a = [1, 2]
b = [1, 2]
```

执行：

```python
a == b
```

结果：

```python
True
```

因为内容一样。

但是：

```python
a is b
```

结果：

```python
False
```

因为：

```text
a → List A

b → List B
```

是两个不同对象。

---

## 为什么经常写：

```python
x is None
```

而不是：

```python
x == None
```

因为我们判断的是：

> x 是否就是 None 这个对象。

推荐：

```python
if x is None:
    ...
```

---

# 八、浅拷贝和深拷贝

## 直接赋值

```python
a = [1, 2, 3]

b = a
```

没有复制。

只是：

```text
a ──┐
    ↓
  List
    ↑
b ──┘
```

---

## Shallow Copy

```python
b = a.copy()
```

创建一个新的外层 List。

例如：

```python
a = [[1, 2], [3, 4]]

b = a.copy()
```

结构类似：

```text
a → Outer List A ──→ Inner List 1
                 └─→ Inner List 2

b → Outer List B ──→ Inner List 1
                 └─→ Inner List 2
```

外层不同。

内部对象还是共享。

所以：

```python
b[0].append(100)
```

之后：

```python
print(a)
```

得到：

```python
[[1, 2, 100], [3, 4]]
```

---

## Deep Copy

```python
import copy

b = copy.deepcopy(a)
```

会递归复制内部对象。

因此：

```text
a
和
b
```

基本成为独立的数据结构。

---

# 九、Python 参数传递是什么？

Python 既不能简单理解成传统的：

```text
Pass by Value
```

也不能简单说：

```text
Pass by Reference
```

更准确的说法是：

```text
Object Reference / Call by Sharing
```

例如：

```python
def modify(x):
    x.append(100)

a = [1, 2]

modify(a)
```

结果：

```python
print(a)
# [1, 2, 100]
```

因为：

```text
a
和
x
```

都指向同一个 List。

但是：

```python
def modify(x):
    x = [100]
```

调用：

```python
a = [1, 2]

modify(a)
```

`a` 不会变。

因为：

```text
x = [100]
```

只是让局部变量 `x` 指向了另一个对象。

---

# 十、*args 是什么？

假设：

```python
def add(a, b):
    return a + b
```

只能接受两个参数。

如果参数数量不确定：

```python
def add(*args):
    print(args)
```

调用：

```python
add(1, 2, 3, 4)
```

得到：

```python
(1, 2, 3, 4)
```

所以：

```text
*args
```

把多个：

```text
位置参数
```

收集成：

```text
tuple
```

例如：

```python
def add(*args):
    return sum(args)
```

---

# 十一、**kwargs 是什么？

`kwargs`：

```text
Keyword Arguments
```

例如：

```python
def show(**kwargs):
    print(kwargs)
```

调用：

```python
show(
    name="Alice",
    age=25
)
```

得到：

```python
{
    "name": "Alice",
    "age": 25
}
```

所以：

```text
**kwargs
```

把多个：

```text
keyword arguments
```

收集成：

```text
dict
```

---

# 十二、为什么 Agent 代码经常出现 **arguments？

例如 LLM 返回：

```python
arguments = {
    "city": "Hangzhou"
}
```

Tool：

```python
def get_weather(city):
    ...
```

可以：

```python
get_weather(**arguments)
```

等价于：

```python
get_weather(city="Hangzhou")
```

这就是为什么 Agent Runtime 经常写：

```python
tool(**tool_call["arguments"])
```

完整逻辑：

```text
LLM

{
  "tool": "weather",
  "arguments": {
      "city": "Hangzhou"
  }
}

        ↓

Agent Runtime

        ↓

get_weather(**arguments)

        ↓

get_weather(city="Hangzhou")
```

---

# 十三、Lambda 是什么？

Lambda：

```text
匿名函数
```

普通函数：

```python
def add(a, b):
    return a + b
```

Lambda：

```python
lambda a, b: a + b
```

基本格式：

```python
lambda 参数: 表达式
```

可以理解：

```text
输入
 ↓
计算
 ↓
自动返回
```

例如：

```python
lambda x: x * 2
```

相当于：

```python
def double(x):
    return x * 2
```

---

# 十四、Lambda 最常见的使用场景：排序

例如：

```python
students = [
    ("Alice", 90),
    ("Bob", 75),
    ("Tom", 85)
]
```

按照成绩：

```python
sorted(
    students,
    key=lambda x: x[1]
)
```

其中：

```python
lambda x: x[1]
```

就是：

```text
("Alice", 90)
      ↓
      90
```

告诉 `sorted`：

> 使用每个元素的第二项作为排序依据。

---

# 十五、sort 和 sorted 有什么区别？

## list.sort()

直接修改原 List：

```python
nums = [3, 1, 2]

nums.sort()

print(nums)
```

结果：

```python
[1, 2, 3]
```

---

## sorted()

返回一个新的 List：

```python
nums = [3, 1, 2]

result = sorted(nums)
```

此时：

```text
nums
仍然是

[3, 1, 2]
```

而：

```text
result

[1, 2, 3]
```

另外 `sorted()` 可以接受很多 Iterable。

---

# 十六、手写 Sort：冒泡排序

```python
def my_sort(nums):

    n = len(nums)

    for i in range(n):

        for j in range(n - 1 - i):

            if nums[j] > nums[j + 1]:

                nums[j], nums[j + 1] = nums[j + 1], nums[j]

    return nums
```

原理：

```text
相邻元素比较
↓
大的往右移动
↓
每轮确定一个最大值
```

例如：

```text
5 2 4 1

↓

2 5 4 1

↓

2 4 5 1

↓

2 4 1 5
```

第一轮之后：

```text
5
```

已经到最后。

时间复杂度：

```text
O(n²)
```

空间复杂度：

```text
O(1)
```

---

# 十七、List Comprehension

普通写法：

```python
result = []

for x in range(10):

    if x % 2 == 0:

        result.append(x * 2)
```

列表推导：

```python
result = [
    x * 2
    for x in range(10)
    if x % 2 == 0
]
```

可以理解：

```text
遍历 x
↓
判断条件
↓
产生新元素
```

---

# 十八、map

`map`：

> 对每一个元素执行一个函数。

例如：

```python
nums = [1, 2, 3]

result = map(
    lambda x: x * 2,
    nums
)
```

得到：

```text
1 → 2
2 → 4
3 → 6
```

转换：

```python
list(result)
```

得到：

```python
[2, 4, 6]
```

---

# 十九、filter

`filter`：

> 根据条件保留元素。

例如：

```python
nums = [1, 2, 3, 4, 5, 6]

result = filter(
    lambda x: x % 2 == 0,
    nums
)
```

得到：

```python
[2, 4, 6]
```

因为：

```text
1 → False
2 → True
3 → False
4 → True
```

---

# 二十、Iterator 是什么？

Iterator：

```text
迭代器
```

可以逐个产生元素。

例如：

```python
nums = [1, 2, 3]

it = iter(nums)
```

然后：

```python
next(it)
# 1

next(it)
# 2

next(it)
# 3
```

继续：

```python
next(it)
```

抛出：

```text
StopIteration
```

---

# 二十一、Iterable 和 Iterator 区别

Iterable：

```text
可以被遍历的对象
```

例如：

```text
list
tuple
dict
set
str
```

Iterator：

```text
可以通过 next()
逐个取值的对象
```

一般：

```text
Iterable
 ↓
iter()
 ↓
Iterator
 ↓
next()
 ↓
element
```

---

# 二十二、Generator 是什么？

Generator：

```text
生成器
```

是一种特殊 Iterator。

普通函数：

```python
def get_nums():

    return [1, 2, 3]
```

一次创建整个 List。

Generator：

```python
def get_nums():

    yield 1
    yield 2
    yield 3
```

调用：

```python
g = get_nums()
```

然后：

```python
next(g)
# 1

next(g)
# 2
```

---

# 二十三、yield 和 return 的区别

`return`：

```text
返回结果
+
函数结束
```

`yield`：

```text
返回一个值
+
暂停函数
+
保存当前状态
+
下次继续执行
```

例如：

```python
def counter():

    yield 1
    yield 2
    yield 3
```

过程：

```text
next()
↓
yield 1
↓
暂停

next()
↓
从上次的位置继续
↓
yield 2
```

---

# 二十四、为什么 Generator 省内存？

例如：

```python
nums = [
    x
    for x in range(10_000_000)
]
```

需要一次创建：

```text
1000万个数字
```

但是：

```python
nums = (
    x
    for x in range(10_000_000)
)
```

Generator 不会一次创建全部数据。

而是：

```text
需要一个
↓
产生一个
```

所以非常适合：

```text
大文件
数据流
大量数据
Streaming
```

---

# 二十五、Decorator 是什么？

Decorator：

> 在不修改原函数代码的情况下，为函数增加功能。

例如：

```python
def log(func):

    def wrapper():

        print("before")

        result = func()

        print("after")

        return result

    return wrapper
```

原函数：

```python
@log
def hello():
    print("hello")
```

调用：

```python
hello()
```

实际类似：

```text
wrapper()
↓
before
↓
hello()
↓
after
```

---

# 二十六、Decorator 本质是什么？

这：

```python
@log
def hello():
    ...
```

基本可以理解为：

```python
hello = log(hello)
```

核心利用：

```text
Python 中函数是一等对象
```

函数可以：

```text
赋值给变量
作为参数
作为返回值
```

---

# 二十七、为什么 FastAPI 经常出现 Decorator？

例如：

```python
@app.get("/users")
def get_users():
    ...
```

可以理解为：

```text
把 get_users 函数
注册到

GET /users
```

因此 Decorator 在 Python Backend 非常常见。

---

# 二十八、try / except / else / finally

基本结构：

```python
try:

    risky_operation()

except ValueError:

    print("error")

else:

    print("success")

finally:

    print("always run")
```

含义：

```text
try
→ 尝试执行

except
→ 出异常执行

else
→ 没有异常执行

finally
→ 无论是否异常通常都会执行
```

---

# 二十九、Agent Tool 为什么必须做异常处理？

例如：

```python
def get_weather(city):

    return requests.get(...)
```

可能：

```text
Timeout
API Error
Invalid Parameter
Network Error
```

所以应该：

```python
try:

    result = get_weather(city)

except TimeoutError:

    result = {
        "error": "tool timeout"
    }
```

然后 Agent 决定：

```text
Retry
Fallback
Ask User
Return Error
```

而不是整个 Agent 崩溃。

---

# 三十、with 是什么？

例如：

```python
with open("data.txt") as f:

    data = f.read()
```

离开 `with` 后：

```text
文件自动关闭
```

不用：

```python
f.close()
```

---

# 三十一、Context Manager 是什么？

`with` 背后是：

```text
Context Manager
```

核心协议：

```python
__enter__()

__exit__()
```

可以理解：

```text
进入环境
↓
申请资源
↓
执行代码
↓
离开环境
↓
释放资源
```

除了文件，还经常用于：

```text
Database Connection
Lock
Transaction
Temporary Resource
```

---

# 三十二、OOP：Class 和 Object

定义：

```python
class Agent:

    def __init__(self, model):

        self.model = model

    def run(self, query):

        return self.model(query)
```

创建：

```python
agent = Agent(model)
```

其中：

```text
Agent
= Class

agent
= Object / Instance
```

---

# 三十三、self 是什么？

例如：

```python
class Agent:

    def __init__(self, name):

        self.name = name
```

这里：

```text
self
```

表示：

```text
当前这个实例
```

例如：

```python
a = Agent("A")
b = Agent("B")
```

那么：

```text
a.name = A
b.name = B
```

---

# 三十四、__init__ 是什么？

创建对象之后，用于初始化对象。

例如：

```python
class Agent:

    def __init__(self, model):

        self.model = model
```

执行：

```python
agent = Agent("GPT")
```

就会初始化：

```text
agent.model = "GPT"
```

严格来说：

```text
__new__
负责创建对象

__init__
负责初始化对象
```

---

# 三十五、继承

例如：

```python
class Agent:

    def run(self):
        print("running")
```

子类：

```python
class RAGAgent(Agent):

    def retrieve(self):
        print("retrieving")
```

于是：

```python
agent = RAGAgent()

agent.run()
agent.retrieve()
```

子类可以使用父类功能。

---

# 三十六、super()

例如：

```python
class Agent:

    def __init__(self, model):
        self.model = model
```

子类：

```python
class RAGAgent(Agent):

    def __init__(self, model, retriever):

        super().__init__(model)

        self.retriever = retriever
```

`super()`：

> 调用父类实现。

---

# 三十七、@staticmethod

例如：

```python
class Calculator:

    @staticmethod
    def add(a, b):
        return a + b
```

调用：

```python
Calculator.add(1, 2)
```

不需要：

```text
self
```

因为这个函数和具体实例状态没有关系。

---

# 三十八、@classmethod

例如：

```python
class Agent:

    count = 0

    @classmethod
    def get_count(cls):

        return cls.count
```

这里：

```text
cls
```

代表：

```text
Class
```

而：

```text
self
```

代表：

```text
Instance
```

---

# 三十九、Python 中函数也是对象

这是理解 Agent Tool Registry 很重要的一点。

定义：

```python
def get_weather(city):
    ...
```

可以：

```python
tool = get_weather
```

然后：

```python
tool("Hangzhou")
```

所以函数可以放进 Dict：

```python
TOOLS = {
    "weather": get_weather,
    "search": web_search
}
```

然后：

```python
tool = TOOLS["weather"]

tool("Hangzhou")
```

这就是很多 Agent Tool Registry 的基本原理。

---

# 四十、Agent Tool Registry 如何实现？

例如：

```python
TOOLS = {
    "weather": get_weather,
    "search": web_search,
    "route": get_route
}
```

LLM 返回：

```python
response = {
    "tool": "weather",
    "arguments": {
        "city": "Hangzhou"
    }
}
```

Agent Runtime：

```python
tool_name = response["tool"]

arguments = response["arguments"]

tool = TOOLS[tool_name]

result = tool(**arguments)
```

完整过程：

```text
LLM
↓
tool = weather
arguments = {city: Hangzhou}
↓
Agent Runtime
↓
TOOLS["weather"]
↓
get_weather
↓
get_weather(city="Hangzhou")
↓
Tool Result
```

---

# 四十一、Process 和 Thread

## Process

进程拥有：

```text
独立内存空间
```

例如：

```text
Process A

Process B
```

默认不会直接共享所有内存。

优点：

```text
隔离性强
可以利用多个 CPU Core
```

缺点：

```text
创建和通信成本较高
```

---

## Thread

线程属于同一个 Process。

例如：

```text
Process

├── Thread A
├── Thread B
└── Thread C
```

它们共享：

```text
Memory
```

优点：

```text
创建成本低
共享数据方便
```

但也容易出现：

```text
Race Condition
Lock
Synchronization
```

问题。

---

# 四十二、GIL 是什么？

GIL：

```text
Global Interpreter Lock
```

在人们最常用的 CPython 传统执行模式下，可以简单理解：

> 一个进程中通常只有一个线程在同一时刻执行 Python 字节码。

因此对于：

```text
CPU-intensive
```

例如：

```text
大量纯 Python 数值计算
```

多个 Python Thread 通常不能简单实现真正的 CPU 并行加速。

这种情况通常考虑：

```text
multiprocessing
```

---

# 四十三、为什么 I/O 密集任务仍然适合 Thread？

例如：

```text
HTTP Request
Database
File I/O
LLM API
```

大量时间是在：

```text
等待
```

而不是 CPU 计算。

例如：

```text
Thread A
→ 等 LLM API

CPU 可以执行

Thread B
→ 发送 HTTP Request
```

所以 Thread 对 I/O 密集任务仍然很有价值。

---

# 四十四、Coroutine 是什么？

Coroutine：

```text
协程
```

可以理解成：

> 一个任务在等待 I/O 的时候主动让出执行权，让 Event Loop 去执行其他任务。

例如：

```text
Task A

请求 LLM
↓
等待网络

此时不傻等
↓
执行 Task B
```

---

# 四十五、async / await

定义异步函数：

```python
async def get_weather():

    ...
```

调用异步操作：

```python
result = await get_weather()
```

`await` 可以理解：

```text
这个操作现在需要等待
↓
当前 Coroutine 暂停
↓
Event Loop 可以执行其他 Task
↓
结果回来
↓
继续执行
```

---

# 四十六、为什么 Agent 特别适合 Async？

Agent 大量操作都是：

```text
LLM API
Web Search
Vector DB
Database
MCP
External API
```

它们大多数是：

```text
I/O Bound
```

例如同时调用：

```text
Weather Tool
Search Tool
Route Tool
```

如果串行：

```text
Weather 1s
↓
Search 2s
↓
Route 2s

Total ≈ 5s
```

如果三个互不依赖，可以并发：

```text
Weather ──┐
Search  ──┼→ await
Route   ──┘

Total ≈ 2s
```

所以 Agent Backend 很常见：

```text
asyncio
```

---

# 四十七、asyncio.gather

例如：

```python
import asyncio

results = await asyncio.gather(
    get_weather(),
    web_search(),
    get_route()
)
```

三个 Coroutine 可以并发等待。

非常适合：

```text
Parallel Tool Calling
```

但前提是这些任务之间没有依赖关系。

---

# 四十八、CPU Bound 和 I/O Bound

## CPU Bound

主要时间用于：

```text
CPU Calculation
```

例如：

```text
图像处理
大量纯 Python 数值计算
复杂算法
```

通常考虑：

```text
Process
Native Library
GPU
```

---

## I/O Bound

主要时间用于：

```text
等待外部资源
```

例如：

```text
HTTP
Database
LLM API
Disk
Web Search
```

通常考虑：

```text
Thread
asyncio
```

Agent 系统大量属于：

```text
I/O Bound
```

---

# 四十九、经典 Python 陷阱：Mutable Default Argument

看：

```python
def add(x, arr=[]):

    arr.append(x)

    return arr
```

执行：

```python
print(add(1))
print(add(2))
```

很多人以为：

```text
[1]

[2]
```

实际：

```text
[1]

[1, 2]
```

原因：

> 默认参数是在函数定义时创建的，而不是每次调用重新创建。

正确：

```python
def add(x, arr=None):

    if arr is None:

        arr = []

    arr.append(x)

    return arr
```

---

# 五十、经典代码题：数组去重

简单：

```python
nums = [1, 2, 2, 3, 3]

result = list(set(nums))
```

但是：

```text
不应该依赖它保持原顺序
```

如果要求保持顺序：

```python
seen = set()

result = []

for x in nums:

    if x not in seen:

        seen.add(x)

        result.append(x)
```

为什么使用 Set？

因为：

```text
x in list
→ O(n)

x in set
→ 平均 O(1)
```

---

# 五十一、经典代码题：字符计数

例如：

```python
text = "hello"
```

统计：

```python
count = {}

for char in text:

    if char not in count:

        count[char] = 0

    count[char] += 1
```

得到：

```python
{
    "h": 1,
    "e": 1,
    "l": 2,
    "o": 1
}
```

也可以：

```python
from collections import Counter

Counter("hello")
```

---

# 五十二、经典代码题：Two Sum

题目：

```text
nums = [2, 7, 11, 15]
target = 9
```

找两个数：

```text
2 + 7 = 9
```

暴力：

```text
O(n²)
```

更好的方式：

```python
def two_sum(nums, target):

    seen = {}

    for i, x in enumerate(nums):

        need = target - x

        if need in seen:

            return [seen[need], i]

        seen[x] = i
```

核心：

```text
当前 x

需要：

target - x
```

利用 Dict：

```text
平均 O(1)
```

查询。

总复杂度：

```text
O(n)
```

---

# 五十三、经典代码题：括号匹配

例如：

```text
([]{})
```

使用：

```text
Stack
```

实现：

```python
def valid(s):

    stack = []

    pairs = {
        ")": "(",
        "]": "[",
        "}": "{"
    }

    for char in s:

        if char in "([{":

            stack.append(char)

        else:

            if not stack:
                return False

            if stack.pop() != pairs[char]:
                return False

    return not stack
```

核心：

```text
Last In
First Out
```

所以使用 Stack。

---

# 五十四、Python Agent 岗最重要的一段代码

如果只能真正理解一段 Agent Python 代码，可以理解这个：

```python
TOOLS = {
    "weather": get_weather,
    "search": web_search,
    "route": get_route
}

response = llm(user_query)

if response["tool"]:

    tool_name = response["tool"]

    arguments = response["arguments"]

    tool = TOOLS.get(tool_name)

    if tool is None:

        raise ValueError("Unknown tool")

    try:

        result = tool(**arguments)

    except Exception as e:

        result = {
            "error": str(e)
        }

    final_response = llm(
        user_query,
        tool_result=result
    )
```

这里实际上把很多 Python 高频知识串起来：

```text
dict
↓
Function Object
↓
Tool Registry
↓
**kwargs
↓
Exception Handling
↓
Function Calling
↓
Agent Runtime
```

进一步改成异步：

```python
result = await tool(**arguments)
```

又会涉及：

```text
Coroutine
async
await
Event Loop
```

所以 Python 基础和 Agent 开发其实不是两套完全独立的知识。

---

# 五十五、面试前最应该掌握的主线

如果时间有限，按照这个顺序复习：

```text
第一层

list
tuple
dict
set

↓

第二层

Mutable / Immutable
is / ==
Reference
Shallow / Deep Copy

↓

第三层

Function
lambda
*args
**kwargs

↓

第四层

Iterator
Generator
yield
Decorator

↓

第五层

Class
self
__init__
inheritance
staticmethod
classmethod

↓

第六层

Exception
with
Context Manager

↓

第七层

Process
Thread
GIL
Coroutine
async
await

↓

第八层

Agent Runtime
Tool Registry
Function Calling
Parallel Tool Calling
```

---

# 五十六、AI Agent 岗 Python 一句话速记

```text
list
→ 有序、可变

tuple
→ 有序、不可变

set
→ 去重、平均 O(1) 查找

dict
→ Hash Table、Key-Value、平均 O(1) 查找

== 
→ 比较值

is
→ 比较对象身份

copy
→ 浅拷贝

deepcopy
→ 递归深拷贝

*args
→ 多个位置参数 → tuple

**kwargs
→ 多个关键字参数 → dict

lambda
→ 小型匿名函数

iterator
→ next() 逐个取值

generator
→ 特殊 iterator，按需生成

yield
→ 返回一个值并暂停函数状态

decorator
→ 不修改原函数主体而扩展行为

self
→ 当前实例

classmethod
→ 操作 Class

staticmethod
→ 与 Instance 状态无关的普通函数

try / except
→ 异常处理

with
→ Context Manager 管理资源生命周期

Process
→ 独立内存，适合 CPU 并行等场景

Thread
→ 共享内存，适合很多 I/O 场景

GIL
→ CPython 中限制多个线程同时执行 Python 字节码的机制

Coroutine
→ 可暂停和恢复的任务

async
→ 定义 Coroutine

await
→ 等待异步操作时让出执行权

Agent Runtime
→ 编排 LLM、State 和 Tools

Tool Registry
→ Tool Name 到 Python Function 的映射

Function Calling
→ LLM 生成结构化 Tool Call

Tool Execution
→ Python Runtime 真正执行函数
```

---

# 五十七、最终理解

对于 AI Agent 开发，Python 最重要的不是背语法，而是理解：

```text
Python Function
        ↓
Function Object
        ↓
Tool Registry
        ↓
LLM Function Calling
        ↓
Agent Runtime
        ↓
Python Function Execution
        ↓
Tool Result
        ↓
LLM
```

例如：

```python
TOOLS = {
    "weather": get_weather
}
```

LLM：

```json
{
    "tool": "weather",
    "arguments": {
        "city": "Hangzhou"
    }
}
```

Python：

```python
tool = TOOLS["weather"]

result = tool(
    city="Hangzhou"
)
```

如果参数来自 Dict：

```python
result = tool(**arguments)
```

如果 Tool 是异步的：

```python
result = await tool(**arguments)
```

如果多个 Tool 可以并行：

```python
results = await asyncio.gather(
    tool_a(),
    tool_b(),
    tool_c()
)
```

这条链真正理解之后：

```text
dict
function
lambda
**kwargs
exception
async / await
```

这些看起来分散的 Python 面试知识，就会和 Agent 开发连成一套完整体系。