---
title: "Augmented Assignment Madness"
date: 2018-05-30T01:53:03-05:00
draft: false
---

Python's augmented assignment behavior bothers me.\
`x += y` is currently equivalent to \
`x = x.__iadd__(y)`.
This has some strange implications.
From the [Python 3 Programming FAQ], we have:

```none
>>> nums = ([0], [3,4])
>>> nums[0].extend([1])
>>> nums[0] += [2]
Traceback (most recent call last):
	File "<stdin>", line 1, in <module>
TypeError: 'tuple' object does not support item assignment
>>> nums[0]
[0, 1, 2]
```

And a strange example, due to lack of `return self`:
```py
class Foo:
	def __init__(self, n=0):
		self.n = n
	
	def __iadd__(self, m):
		self.n += m
		# note: most built in classes `return self` here

>>> x = Foo()
>>> x += 1
>>> x is None
True
```

```py
def inc(x):
	x += 1
	return x is None

>>> x = Foo()
>>> inc(x)
True
>>> x is None
False
```

Requiring Python programmers to `return self` at the end of every augmented assignment method seems… weird.
If a method has side effects, it should not return.

Personally, I think that `x += y` should not reassign, *if x has a `__iadd__` method*.
This would remove all of the above subtleties in cases where x has an `__iadd__`,
but would make it inconsistent with immutable objects which do not have an `__iadd__`.
In that case, CPython falls back to `x = x.__add__(y)`, which does reassign.


[Python 3 Programming FAQ]: https://docs.python.org/3/faq/programming.html#why-does-a-tuple-i-item-raise-an-exception-when-the-addition-works
