---
title: "Python Interview Prep Guide Tcs"
category: "Interview Prep"
date_updated: 2026-08-06
---

> **Note:** Continuous reading edition. Optimized for quick scanning, native code rendering, and distraction-free learning.

---

**PYTHON INTERVIEW PREPARATION GUIDE**

*Theory Questions + Coding Questions — Focused for TCS Technical Round 2*

Compiled from real TCS interview experiences plus common cross-company Python rounds (2025-2026)

*Prepared for: Prashant \| Round 1 cleared — Round 2 tomorrow \| July 2026*

PART 1 — THEORY / CONCEPTUAL QUESTIONS

TCS Round 2 is the technical face-to-face round. Expect a mix of Python fundamentals, OOP, and 1-2 short hand-written/whiteboard coding problems, often blended with questions about your resume and project experience.

## 1.1 Python Fundamentals

### Q1. Is Python interpreted or compiled? *\[Asked at: Freshers/NQT round, asked everywhere\]*

**A:** Python source (.py) is first compiled into bytecode (.pyc), then the Python Virtual Machine (PVM) interprets and executes that bytecode line by line. So Python is technically both compiled and interpreted — CPython (the standard implementation) does both steps automatically.

### Q2. What are Python's key features? *\[Asked at: TCS NQT & technical rounds, entry-level\]*

**A:** Simple, readable syntax; interpreted and dynamically typed; supports multiple paradigms (procedural, OOP, functional); huge standard library; automatic memory management (reference counting + garbage collection); portable across platforms; extensible via C/C++.

### Q3. What is the difference between a list, a tuple, and a set? *\[Asked at: Very common — TCS, Infosys, all companies\]*

**A:** A list is ordered and mutable — items can be changed, added, or removed. A tuple is ordered but immutable — once created it cannot be changed, which makes it faster and usable as a dictionary key. A set is unordered, mutable, and only stores unique elements (no duplicates).

### Q4. What is the difference between is and ==? *\[Asked at: Common trick question\]*

**A:** == compares values for equality. is compares identity — whether both variables point to the exact same object in memory. Two lists with identical contents are == but not is, unless they're literally the same object.

### Q5. What is the difference between deep copy and shallow copy? *\[Asked at: TCS Python interview — very frequently asked\]*

**A:** A shallow copy (copy.copy()) creates a new outer object but keeps references to the same nested/inner objects, so changing a nested object affects both copies. A deep copy (copy.deepcopy()) recursively duplicates everything, including nested objects, so the two copies are fully independent.

### Q6. Are strings mutable or immutable in Python? What does that mean practically? *\[Asked at: Common follow-up\]*

**A:** Strings are immutable — any operation that appears to 'modify' a string (like concatenation) actually creates a new string object in memory. This is why repeatedly concatenating strings in a loop is inefficient; using ''.join() or a list buffer is preferred.

### Q7. What is PEP 8? *\[Asked at: TCS process-oriented question\]*

**A:** PEP 8 is Python's official style guide — covering naming conventions, indentation (4 spaces), line length, import ordering, and other conventions that keep Python code consistent and readable across projects.

## 1.2 Data Structures & Built-ins

### Q8. What is the difference between append() and extend() on a list? *\[Asked at: TCS, basic-to-mid level\]*

**A:** append() adds its argument as a single element to the end of the list (even if that argument is itself a list, it's nested as one item). extend() takes an iterable and adds each of its elements individually to the list.

### Q9. How do dictionaries work internally in Python? *\[Asked at: Mid-to-senior TCS rounds\]*

**A:** A dictionary is implemented as a hash table: each key is hashed to determine its storage slot, giving average O(1) lookup, insert, and delete. Since Python 3.7, dictionaries also maintain insertion order as an implementation detail (later made an official language guarantee).

### Q10. What happens if you try to access a dictionary key that doesn't exist? *\[Asked at: Common practical question\]*

**A:** It raises a KeyError. You can avoid this using dict.get(key, default) which returns a default value instead of raising, using dict.setdefault(), or wrapping the access in a try/except KeyError block, or checking 'if key in dict' first.

### Q11. What is a frozenset and why would you use one? *\[Asked at: Mid-level\]*

**A:** A frozenset is the immutable version of a set — it can't be modified after creation, which makes it hashable and usable as a dictionary key or as an element of another set, unlike a regular mutable set.

### Q12. What is List Comprehension and why use it? *\[Asked at: Every Python interview\]*

**A:** A concise syntax to build a new list (or dict/set) from an iterable in a single readable line, optionally with a filter condition, e.g. \[x\*\*2 for x in nums if x % 2 == 0\]. It's typically more readable and often faster than an equivalent explicit for-loop with .append().

### Q13. What is the difference between \*args and \*\*kwargs? *\[Asked at: Very frequently asked\]*

**A:** \*args collects any number of extra positional arguments into a tuple inside the function. \*\*kwargs collects any number of extra keyword arguments into a dictionary. Both let a function accept a flexible, variable number of arguments.

## 1.3 Functions, Scope & Functional Concepts

### Q14. What is the difference between a local and a global variable? *\[Asked at: TCS Round 2 basics\]*

**A:** A local variable is created inside a function and only exists/is accessible within that function's scope. A global variable is defined at the top module level and is accessible everywhere, though you must use the 'global' keyword inside a function if you intend to modify it there.

### Q15. What is a lambda function? When would you use one? *\[Asked at: Every Python round\]*

**A:** A lambda is a small anonymous, single-expression function defined inline, e.g. lambda x: x \* 2. It's typically used for short throwaway functions passed to higher-order functions like map(), filter(), or sorted(key=...), where defining a full named function would be overkill.

### Q16. What is the difference between map(), filter(), and reduce()? *\[Asked at: Common functional-programming question\]*

**A:** map() applies a function to every item of an iterable and returns the transformed results. filter() keeps only the items for which a function returns True. reduce() (from functools) cumulatively applies a function to the items to reduce the iterable to a single value, like a running total.

### Q17. What are decorators and why are they useful? *\[Asked at: Mid-to-senior — very common\]*

**A:** A decorator is a function that wraps another function to extend or modify its behavior without changing its source code, applied using the @decorator_name syntax. Common uses include logging, timing, authentication checks, and caching/memoization.

### Q18. What is the difference between return and yield? *\[Asked at: Common, tests generator understanding\]*

**A:** return exits the function immediately and sends back a single value, ending the function's execution. yield pauses the function, sends back a value, and preserves the function's state so it can resume from exactly that point on the next call — this is what turns a function into a generator.

### Q19. What are generators and why are they memory-efficient? *\[Asked at: TCS, very commonly asked\]*

**A:** Generators are functions that use yield to produce values one at a time, lazily, instead of building and returning an entire collection in memory at once. This makes them ideal for processing large datasets or infinite sequences without high memory overhead.

### Q20. What is a closure in Python? *\[Asked at: Mid-to-senior\]*

**A:** A closure is a nested (inner) function that remembers and has access to variables from its enclosing (outer) function's scope, even after the outer function has finished executing. Closures are the mechanism decorators are typically built on.

## 1.4 Object-Oriented Programming

### Q21. How does Python implement access specifiers (private/public) since it has no explicit keywords? *\[Asked at: TCS Python interview — commonly asked\]*

**A:** Python relies on naming conventions rather than enforced keywords: a single underscore prefix (\_var) signals 'protected, internal use' by convention; a double underscore prefix (\_\_var) triggers name mangling, making it harder (not impossible) to access from outside the class, approximating private.

### Q22. What is the purpose of the self keyword? *\[Asked at: Every OOP-based Python interview\]*

**A:** self refers to the current instance of the class and is used to access that instance's attributes and methods. It must be the first parameter of any instance method, though Python passes it automatically when the method is called on an object.

### Q23. What is the difference between a classmethod and a staticmethod? *\[Asked at: Frequently asked at mid-level\]*

**A:** A classmethod (decorated with @classmethod) receives the class itself (cls) as its first argument and can access/modify class-level state; it's often used for alternative constructors. A staticmethod (@staticmethod) receives neither self nor cls — it behaves like a plain function that's just logically grouped inside the class namespace.

### Q24. Explain the four pillars of OOP with Python examples. *\[Asked at: Very common at TCS\]*

**A:** Encapsulation: bundling data and methods together, restricting direct access via naming conventions or properties. Inheritance: a child class reuses/extends a parent class's attributes and methods. Polymorphism: the same method name behaves differently depending on the object calling it (e.g., overriding a method in a subclass). Abstraction: hiding implementation details behind a simpler interface, often using abstract base classes (the abc module).

### Q25. What is Multiple Inheritance and what is the MRO (Method Resolution Order)? *\[Asked at: Mid-to-senior\]*

**A:** Multiple Inheritance is when a class inherits from more than one parent class. MRO is the order Python follows to look up a method or attribute across that inheritance chain, computed using the C3 linearization algorithm — visible via ClassName.\_\_mro\_\_.

### Q26. What are magic/dunder methods? Give examples. *\[Asked at: Senior-level Python rounds\]*

**A:** Special methods surrounded by double underscores that let objects integrate with Python's built-in syntax and functions: \_\_init\_\_ (constructor), \_\_str\_\_/\_\_repr\_\_ (string representation), \_\_len\_\_, \_\_eq\_\_, \_\_add\_\_ (operator overloading), \_\_iter\_\_/\_\_next\_\_ (making an object iterable).

## 1.5 Exception Handling & Memory Management

### Q27. How does exception handling work in Python (try/except/else/finally)? *\[Asked at: Asked at every level\]*

**A:** try holds code that might raise an error. except catches and handles specific exception types. else runs only if no exception occurred. finally always runs regardless of whether an exception occurred, typically used for cleanup like closing files or connections.

### Q28. What is the difference between an Exception and an Error in Python? *\[Asked at: Conceptual, mid-level\]*

**A:** BaseException is the root; Exception is the class most user-handleable problems derive from (ValueError, KeyError, etc.) and should generally be caught. Error-type issues like SystemExit or KeyboardInterrupt derive directly from BaseException and typically should NOT be silently caught.

### Q29. How does Python manage memory? *\[Asked at: Common at TCS mid-level\]*

**A:** Python uses automatic memory management via a private heap: reference counting tracks how many references point to an object, and once that count hits zero the memory is freed immediately. A cyclic garbage collector additionally detects and cleans up reference cycles (e.g., two objects referencing each other) that reference counting alone can't catch.

### Q30. What is the Global Interpreter Lock (GIL) and how does it affect multithreading? *\[Asked at: Very frequently asked at TCS and elsewhere\]*

**A:** The GIL is a mutex in CPython that allows only one thread to execute Python bytecode at a time, even on multi-core machines. This means Python threads don't give a true speed-up for CPU-bound work, but they still work well for I/O-bound tasks (file/network operations) since the GIL is released during I/O waits. For CPU-bound parallelism, the multiprocessing module (separate processes, separate GILs) is used instead.

## 1.6 Modules Frequently Mentioned at TCS

### Q31. What is the difference between NumPy and SciPy? *\[Asked at: TCS mid-level Python developer role\]*

**A:** NumPy provides the core building blocks — fast, memory-efficient N-dimensional arrays and basic array operations. SciPy builds on top of NumPy, adding higher-level scientific computing functionality like optimization, integration, interpolation, and signal processing.

### Q32. What is the difference between an iterator and an iterable? *\[Asked at: Common at TCS\]*

**A:** An iterable is any object you can loop over (implements \_\_iter\_\_), like a list or string. An iterator is the object actually produced by calling iter() on an iterable — it implements both \_\_iter\_\_ and \_\_next\_\_, and remembers its current position as you call next() on it.

# PART 2 — CODING / PROGRAM-WRITING QUESTIONS

In TCS Round 2 you'll typically be asked to write 1-3 short programs by hand or in a shared editor. These are the patterns that repeat most often across TCS and general Python technical rounds.

## 2.1 String Problems

### Q1. Check if a string is a palindrome. *\[Asked at: TCS, asked at almost every level\]*

**A:** Compare the string to its reverse using slicing.

```python
def is_palindrome(s):
s = s.lower().replace(' ', '')
return s == s[::-1]
print(is_palindrome("Madam")) # True
print(is_palindrome("Hello")) # False
```

### Q2. Reverse a string without using the built-in reversed() or slicing shortcuts (show the logic). *\[Asked at: TCS Round 2 favorite\]*

**A:** Manually build the reversed string by walking from the end.

```python
def reverse_string(s):
result = ''
for ch in s:
result = ch + result
return result
print(reverse_string("Python")) # nohtyP
```

### Q3. Count the frequency of each character in a string. *\[Asked at: Very common — TCS, Infosys\]*

**A:** Use a dictionary to tally occurrences in a single pass.

```python
def char_frequency(s):
freq = {}
for ch in s:
freq[ch] = freq.get(ch, 0) + 1
return freq
print(char_frequency("banana"))
# {'b': 1, 'a': 3, 'n': 2}
```

### Q4. Find the first non-repeating character in a string. *\[Asked at: Common follow-up to frequency counting\]*

**A:** Build a frequency map first, then scan the string again in order to find the first character with count 1.

```python
def first_unique_char(s):
freq = {}
for ch in s:
freq[ch] = freq.get(ch, 0) + 1
for ch in s:
if freq[ch] == 1:
return ch
return None
print(first_unique_char("swiss")) # 'w'
```

### Q5. Check if two strings are anagrams of each other. *\[Asked at: Very frequently asked\]*

**A:** Sorting both strings and comparing is the simplest approach; character-count comparison is the O(n) alternative.

```python
def is_anagram(s1, s2):
return sorted(s1) == sorted(s2)
print(is_anagram("listen", "silent")) # True
```

### Q6. Check if a string (of brackets) has balanced parentheses. *\[Asked at: Classic stack problem, common at TCS\]*

**A:** Use a stack: push opening brackets, and on a closing bracket check it matches the top of the stack.

```python
def is_balanced(s):
stack = []
pairs = {')': '(', ']': '[', '}': '{'}
for ch in s:
if ch in '([{':
stack.append(ch)
elif ch in ')]}':
if not stack or stack.pop() != pairs[ch]:
return False
return not stack
print(is_balanced("{[()]}")) # True
print(is_balanced("{[(])}")) # False
```

### Q7. Count the number of words in a sentence/file. *\[Asked at: TCS file-handling question\]*

**A:** split() on whitespace gives a quick word count.

```sql
def word_count(text):
return len(text.split())
print(word_count("TCS interview tomorrow")) # 3
# From a file:
with open('sample.txt') as f:
text = f.read()
print(word_count(text))
```

## 2.2 List & Array Problems

### Q8. Find the second largest number in a list. *\[Asked at: TCS Python interview — extremely common\]*

**A:** Track the largest and second largest in a single pass, without sorting.

```python
def second_largest(nums):
first = second = float('-inf')
for n in nums:
if n > first:
second = first
first = n
elif first > n > second:
second = n
return second
print(second_largest([4, 1, 7, 7, 3])) # 4
```

### Q9. Remove duplicates from a list while preserving order. *\[Asked at: Very frequently asked\]*

**A:** Use a set to track what's been seen, but build the result as a list to preserve original order (a plain set() would lose order).

```python
def remove_duplicates(lst):
seen = set()
result = []
for item in lst:
if item not in seen:
seen.add(item)
result.append(item)
return result
print(remove_duplicates([1, 2, 2, 3, 1, 4])) # [1, 2, 3, 4]
```

### Q10. Find common elements between two lists. *\[Asked at: Common, sometimes asked 'without using set()'\]*

**A:** The Pythonic way uses set intersection; the manual way (sometimes explicitly requested) uses a list comprehension.

```text
a = [1, 2, 3, 4]
b = [3, 4, 5, 6]
# Using sets
print(list(set(a) & set(b))) # [3, 4]
# Without sets
print([x for x in a if x in b]) # [3, 4]
```

### Q11. Find the maximum difference (j - i) such that array\[j\] \> array\[i\], with j \> i. *\[Asked at: GfG/DataCamp-style, asked in mid-level rounds\]*

**A:** Track the minimum value seen so far and the best difference found while scanning left to right.

```python
def max_diff(arr):
min_val = arr[0]
max_d = -1
for j in range(1, len(arr)):
if arr[j] > min_val:
max_d = max(max_d, arr[j] - min_val)
else:
min_val = arr[j]
return max_d
print(max_diff([2, 3, 10, 6, 4, 8, 1])) # 8 (10 - 2)
```

### Q12. Find two numbers in a list that add up to a target (Two Sum). *\[Asked at: Extremely common across all companies\]*

**A:** Use a dictionary to remember numbers already seen and their index; for each new number, check if its complement is already in the dictionary.

```python
def two_sum(nums, target):
seen = {}
for i, n in enumerate(nums):
complement = target - n
if complement in seen:
return [seen[complement], i]
seen[n] = i
return None
print(two_sum([2, 7, 11, 15], 9)) # [0, 1]
```

### Q13. Flatten a nested list. *\[Asked at: Common data-structure question\]*

**A:** Use recursion to handle arbitrary nesting depth.

```python
def flatten(lst):
result = []
for item in lst:
if isinstance(item, list):
result.extend(flatten(item))
else:
result.append(item)
return result
print(flatten([1, [2, 3, [4, 5]], 6])) # [1, 2, 3, 4, 5, 6]
```

### Q14. Sort a list of dictionaries by a specific key. *\[Asked at: Common practical question\]*

**A:** Use sorted() with a key function/lambda.

```text
employees = [{'name': 'A', 'age': 30}, {'name': 'B', 'age': 25}]
sorted_emp = sorted(employees, key=lambda x: x['age'])
print(sorted_emp)
# [{'name': 'B', 'age': 25}, {'name': 'A', 'age': 30}]
```

## 2.3 Dictionary, OOP & Program-Design Problems

### Q15. Merge two dictionaries. *\[Asked at: Very common, has multiple valid approaches\]*

**A:** Python 3.9+ supports the \| merge operator; older versions use unpacking or .update().

```sql
d1 = {'a': 1, 'b': 2}
d2 = {'b': 3, 'c': 4}
# Python 3.9+
merged = d1 | d2
print(merged) # {'a': 1, 'b': 3, 'c': 4}
# Compatible with older versions
merged2 = {**d1, **d2}
```

### Q16. Write a Python program to implement a simple calculator using functions. *\[Asked at: TCS Round 2 favorite for testing function design\]*

**A:** Define one function per operation and dispatch based on user choice.

```python
def add(a, b): return a + b
def subtract(a, b): return a - b
def multiply(a, b): return a * b
def divide(a, b): return a / b if b != 0 else 'Error: division by zero'
def calculator(a, b, op):
ops = {'+': add, '-': subtract, '*': multiply, '/': divide}
return ops[op](a, b)
print(calculator(10, 5, '+')) # 15
```

### Q17. Implement a simple Stack and Queue using a Python list. *\[Asked at: Tests understanding of data structures via OOP\]*

**A:** A Stack uses append()/pop() from the end (LIFO). A Queue uses append() at the end and pop(0) from the front (FIFO), though collections.deque is preferred for real use since pop(0) on a list is O(n).

```python
class Stack:
def __init__(self):
self.items = []
def push(self, item):
self.items.append(item)
def pop(self):
return self.items.pop() if self.items else None
def peek(self):
return self.items[-1] if self.items else None
s = Stack()
s.push(1); s.push(2)
print(s.pop()) # 2
```

### Q18. Write a class hierarchy demonstrating inheritance and method overriding. *\[Asked at: Core OOP coding question at TCS\]*

**A:** A subclass overrides the parent's method to provide specialized behavior, while still being usable wherever the parent type is expected (polymorphism).

```python
class Animal:
def speak(self):
return "Some generic sound"
class Dog(Animal):
def speak(self):
return "Woof!"
class Cat(Animal):
def speak(self):
return "Meow!"
for a in [Dog(), Cat(), Animal()]:
print(a.speak())
```

### Q19. Write a decorator that logs the execution time of a function. *\[Asked at: Mid-to-senior, tests decorators practically\]*

**A:** Wrap the function call with time.time() calls before and after, and print the elapsed duration.

```python
import time
def timer(func):
def wrapper(*args, **kwargs):
start = time.time()
result = func(*args, **kwargs)
end = time.time()
print(f"{func.__name__} took {end - start:.4f}s")
return result
return wrapper
@timer
def slow_add(a, b):
time.sleep(1)
return a + b
slow_add(2, 3)
```

### Q20. Write a generator function that yields Fibonacci numbers up to n terms. *\[Asked at: Common generator/yield question\]*

**A:** Use yield inside a loop to lazily produce each Fibonacci number without storing the whole sequence in memory.

```python
def fibonacci(n):
a, b = 0, 1
for _ in range(n):
yield a
a, b = b, a + b
print(list(fibonacci(8)))
# [0, 1, 1, 2, 3, 5, 8, 13]
```

### Q21. Use try/except/finally to handle a divide-by-zero error gracefully. *\[Asked at: TCS basic exception-handling check\]*

**A:** Catch the specific exception type rather than a bare except, and use finally for guaranteed cleanup/messaging.

```python
def safe_divide(a, b):
try:
return a / b
except ZeroDivisionError:
return "Error: Cannot divide by zero"
finally:
print("Operation attempted.")
print(safe_divide(10, 0))
```

## 2.4 Number / Logic Problems

### Q22. Check whether a number is prime. *\[Asked at: Extremely common at TCS and every fresher round\]*

**A:** Only check divisibility up to the square root of n for efficiency.

```python
def is_prime(n):
if n < 2:
return False
for i in range(2, int(n ** 0.5) + 1):
if n % i == 0:
return False
return True
print(is_prime(17)) # True
print(is_prime(18)) # False
```

### Q23. Find the factorial of a number, both iteratively and recursively. *\[Asked at: TCS staple question\]*

**A:** Show both approaches — interviewers often ask you to convert one into the other on the spot.

```python
def factorial_iterative(n):
result = 1
for i in range(2, n + 1):
result *= i
return result
def factorial_recursive(n):
if n <= 1:
return 1
return n * factorial_recursive(n - 1)
print(factorial_iterative(5)) # 120
print(factorial_recursive(5)) # 120
```

### Q24. Check if a number is an Armstrong number. *\[Asked at: Classic TCS NQT/technical round question\]*

**A:** An Armstrong number equals the sum of its own digits each raised to the power of the digit count.

```python
def is_armstrong(n):
digits = str(n)
power = len(digits)
total = sum(int(d) ** power for d in digits)
return total == n
print(is_armstrong(153)) # True (1**3 + 5**3 + 3**3 = 153)
```

### Q25. Print the Fibonacci series up to n terms (without generators, plain loop). *\[Asked at: Fresher/NQT staple\]*

**A:** Iteratively build the sequence using two running variables.

```python
def fibonacci_series(n):
a, b = 0, 1
series = []
for _ in range(n):
series.append(a)
a, b = b, a + b
return series
print(fibonacci_series(10))
```

### Q26. Swap two numbers without using a temporary variable. *\[Asked at: Common quick logic question\]*

**A:** Python allows tuple unpacking to swap values directly in one line.

```text
a, b = 5, 10
a, b = b, a
print(a, b) # 10 5
```

# APPENDIX — Last-Minute TCS Round 2 Notes

## What TCS Round 2 typically covers

- It's a face-to-face technical round testing Python fundamentals, OOP, DBMS/SQL basics, and your resume/project experience together — not just Python in isolation.

- Expect 1-3 short hand-written or shared-editor coding problems (string/list manipulation, simple number logic) rather than heavy DSA/algorithm design.

- Interviewers commonly circle back to your listed projects and ask you to explain design choices — be ready to connect your MDM/RAG/LangGraph project work to core programming concepts if it comes up.

- Some interviewers ask almost no coding at all and focus purely on verbal explanations of OOP, exception handling, and Python behavior — so rehearse explaining concepts out loud, not just writing code silently.

## Fast final-night checklist

- Be able to explain: mutable vs immutable, shallow vs deep copy, \*args/\*\*kwargs, decorators, generators (yield vs return), and the GIL — these repeat constantly in TCS interviews.

- Be able to write from memory, without hesitation: second-largest-in-list, palindrome check, prime check, factorial (iterative + recursive), balanced-parentheses, and two-sum.

- Practice narrating your thought process out loud while coding — TCS interviewers weigh clarity of explanation heavily, not just a working final answer.

- Keep 2-3 project talking points ready (e.g., your SharePoint RAG chatbot or SNOW AI Automation work) in case they ask 'tell me about a Python project you've built' — connect it back to OOP/exception-handling/performance concepts from this guide.

- Get a good night's sleep — TCS interviewers also assess communication and composure, not just syntax recall.
