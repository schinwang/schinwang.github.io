---
bg: "flower.jpeg"
layout: page
title: "pyNotes"
crawlertitle: "python interesting notes"
permalink: /pynotes
summary: ""
active: pynotes
---

##### 1. **[chain comparisons](https://pythontips.com/2017/10/15/weird-comparison-issue-in-python/#)**

Ex1:

```python
>>>a=10;res = 'bingo' if (10<a<20) else 'woops'
```

Ex2:

```python
>>>a = 3; b = False; c = """12"""; d = 4.7;res = (d + 2 * a > int(c) == b)
```

当同一个表达式出现多个比较操作符时, python会按顺序(从左到右)计算各比较操作，然后对其结果取'与'运算

上面两个式子等价于

```python
>>>(10<a) and (a<20)
>>>((d+2*a)>int(c)) and (int(c) == b)
```

