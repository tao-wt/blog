---
title: python 操作符优先级
date: 2023-12-17 19:12:25
index_img: /img/cat-index.jpg
tags:
  - python
categories:
  - python
excerpt: 遇到的操作符问题，及python中操作符优先级说明
---
### 使用python时遇到的运算符优先级问题
---
以下代码在`list_2`为空时返回`else`后面的值，若期望在`list_2`为空时返回`if`后面的值，则需要使用括号。
```python
>>> list_1 = [1, 2, 3]
>>> list_2 = list()
>>> value = list_1 + list_2 if list_2 else ["a", "b"]
>>> value
['a', 'b']
>>>
```

### 其它相关操作符优先级说明
---
Comparisons can be chained. For example, `a < b == c` tests whether a is less than b and moreover b equals c. 

All comparison operators have the same priority, which is lower than that of all numerical数字的 operators. 

Boolean operators and, or and not have lower priorities than comparison operators; between them, not has the highest priority and or the lowest, so that `A and not B or C` is equivalent to `(A and (not B)) or C`. As always, parentheses can be used to express表示 the desired composition作文，构成，成份. 

**Note** that in Python, unlike C, assignment inside expressions must be done explicitly with the walrus海象 operator `:=`. 
```python
>>> if a := "wsx":
...     print(a)
...
wsx
>>> if a := "":
...     print(a)
...
>>>
```