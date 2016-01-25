---
layout: post
title:  "EAFP 和 LBYL"
date:   2016-01-25
categories: Python
---


[EAFP](https://docs.python.org/3/glossary.html#term-eafp)
>Easier to ask for forgiveness than permission. This common Python coding style assumes the existence of valid keys or attributes and catches exceptions if the assumption proves false. This clean and fast style is characterized by the presence of many try and except statements. The technique contrasts with the LBYL style common to many other languages such as C.

[LBYL](https://docs.python.org/3/glossary.html#term-lbyl)
>Look before you leap. This coding style explicitly tests for pre-conditions before making calls or lookups. This style contrasts with the EAFP approach and is characterized by the presence of many if statements.  
In a multi-threaded environment, the LBYL approach can risk introducing a race condition between “the looking” and “the leaping”. For example, the code, if key in mapping: return mapping[key] can fail if another thread removes key from mapping after the test, but before the lookup. This issue can be solved with locks or by using the EAFP approach.

to be continue...


参考:

[EAFP vs. LBYL](http://python.net/~goodger/projects/pycon/2007/idiomatic/handout.html#eafp-vs-lbyl)  
[YouTube: Permission or forgiveness?](https://www.youtube.com/watch?v=AZDWveIdqjY)  
[What is the EAFP principle in Python?](http://stackoverflow.com/questions/11360858/what-is-the-eafp-principle-in-python)  
[Are nested try/except blocks in python a good programming practice?](http://stackoverflow.com/questions/17015230/are-nested-try-except-blocks-in-python-a-good-programming-practice)  
[Do we use try,except in every single function?](http://stackoverflow.com/questions/11478484/do-we-use-try-except-in-every-single-function)  
[海之寻趣: LBYL vs. EAFP](http://www.findfunaax.com/notes/file/155)  
[v2ex: 关于条件检查，异常，错误码的一些思考](http://www.v2ex.com/t/55027)  


