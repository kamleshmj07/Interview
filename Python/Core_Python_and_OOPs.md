# Core Python & OOP Interview Guide
---

## STRUCTURE

This guide provides focused **100% on core Python and OOP** concepts only—no external libraries.

**Study Approach**:
1. Read a topic
2. Code the example
3. Try to explain it aloud
4. Practice with variations

**Success Metric**: You should be able to explain every concept without looking at notes, with real code examples.

---

## QUESTIONS COVERED

1. What is OOP and its 4 principles?
2. Difference between class and object
3. `__init__()` constructor
4. `self` keyword and how it works
5. `super()` function
6. What are "Magic Methods" in Python, and why are they used?
7. What is the difference between `__init__` and `__new__`?
8. How do `__str__` and `__repr__` differ?
9. Which magic method would you use to make an object "callable" like a function?
10. How do you implement the "Context Manager" protocol (the with statement) in a class?
11. What are the magic methods for operator overloading?
12. How can you make a custom object behave like a container (list or dictionary)?
13. What is the `__slots__` magic attribute?
14. What's the Difference Between `is` and `==`?
15. What's the Difference Between Class and Static Methods?
16. Explain Memory Management in Python
17. Mutable vs Immutable in Python
18. Properties and `@property` decorator
19. What is the difference between a Property and a regular Attribute?
20. Why is using `@property` preferred over traditional `get_attr()` and `set_attr()` methods?
21. Can a property be "Read-Only"?
22. What are "Computed Properties"?
23. How does the @property decorator relate to Descriptors?
24. What is the difference between an Iterable and an Iterator?
25. What is a Generator, and how does it differ from a normal function?
26. Why would you use a Generator instead of a List?
27. What is the purpose of the yield keyword?
28. Why should you avoid using mutable default arguments in functions?
29. Are tuples always immutable?
30. What is the difference between a List and a Tuple?
31. How does a Python Dictionary work internally?
32. What are the main characteristics of a Set?
33. What is the difference between `dict.get('key')` and `dict['key']`?
34. What is a Closure in Python?
35.  What is the nonlocal keyword used for in a closure?
36. What is a Decorator?
37. How do you write a decorator that accepts arguments?
38. Can you apply multiple decorators to a single function?
39. What is the difference between an Instance Variable and a Class Variable?
40. What happens if you try to modify a Class Variable through an Instance?
41. When should you use a Class Variable?
42. What is the Method Resolution Order (MRO) in relation to variables?
43. How does the __dict__ attribute differ between a class and its instance?
44. What is an Abstract Base Class (ABC)?
45. Why use ABCs instead of regular inheritance?
46. Can an Abstract Method have code in it?
47. What are "Virtual Subclasses" (the register method)?
48. 
49. 
50. 







## Additional Questions [Yet to be covered]
1. Example/Usecase for Class methods and it's types
2. Explain the "Descriptor Protocol" in more detail, as it is the advanced mechanism that makes properties possible?
3. How properties can be used to protect a "private" variable from being modified by mistake?
4. Decorators concept in python
5. What happens when a Generator or Iterator runs out of items?
6. What are Generator Expressions?
7. What is the `yield from` syntax used for?
8. Can you restart a Generator?
9. What is "String Interning" and how does it relate to immutability?
10. Why can't a list be used as a dictionary key?
11. What is the difference between == and is in the context of mutability?
12. copy vs deepcopy
13. What is the difference between list.append() and list.extend()?
14. What is a "List Comprehension" and why is it used?
15. When should you use a collections.deque instead of a List?
16. How can you merge two dictionaries in Python 3.9+?
17. Complexity Reference (Big O) for list, Set/Dict and Deque
18. Why should you use functools.wraps when writing a decorator?
19. How to create a "Class Decorator" (a class that decorates a function) using the __call__ magic method we discussed earlier?
20. How do you track the total number of instances created for a class?
21. What is the difference between an ABC and an Interface (like in Java)?
22. What are the ABCs in the collections.abc module?
23. 


-----
### INTERMEDIATE LEVEL
1. Inheritance, Multiple inheritance & Diamond problem (C3 linearization)
2. Class methods vs static methods
3. Method overriding
4. Method overloading (using default args)
5. MRO (Method Resolution Order)
6. `@staticmethod` decorator
7. Monkey-patching
8. Duck typing
9. `__new__()` method
10. `global` and `nonlocal` keywords
11. LEGB rule
12. Hashable objects
13. Name mangling (double underscore)
14. Type checking (`isinstance()` vs `type()`)


### SENIOR/LEAD LEVEL
1. Descriptors protocol
2. Metaclasses
3. `__slots__` for memory optimization
4. Reference counting memory model
5. Garbage collection and generational GC
6. Context managers (`with` statement)
7. Iterators and iterables
8. Generators and lazy evaluation
9. Weak references
10. Copy module (shallow vs deep)
11. Performance optimization techniques
12. Circular reference handling
13. Design patterns (Singleton, Factory)
14. Advanced inheritance patterns
15. Memory profiling and optimization
16. Weak references and circular references

### Differentiators (Lead Engineer)
1. Design patterns using Python OOP
2. Writing scalable class hierarchies
3. Memory profiling and optimization
4. Advanced metaclass usage


------------------------------------------------------------------------------

## COMMON INTERVIEW QUESTIONS WITH ANSWERS

### What is OOP and its 4 principles?
**Answer**: Object-Oriented Programming (OOP) is a programming "paradigm" or style that organizes software design around objects and it's associated behaviours. The 4 principles are Encapsulation, Inheritance, Polymorphism, Abstraction.

- **Encapsulation**: This is the practice of keeping attributes and the behaviour bundled together inside a single unit i.e., the object. It also involves "hiding" the internal state of the object.
- **Inheritance**: Inheritance allows one class (a "child") to adopt the attributes and methods of another class (a "parent"). This helps avoid repeating code and promotes reusability of code.
- **Polymorphism**: In Polymorphism, the design allows different behavior/form in different classes under same representation.
- **Abstraction**: Abstraction is about hiding complex implementation details and only showing the necessary features of an object. It reduces complexity by letting you focus on what an object does rather than how it does it.

### Difference between class and object
**Answer** A class is a logical entity; an abstraction. An object is a physical entity that exists in memory at runtime.

### `__init__()` constructor
**Answer** It is the initializer for a class's new instances and is automatically called after an object has been created. It is commonly referred as "Constructor".

### `self` keyword and how it works
**Answer** `self` is a adopted convention for the first parameter of an instance method within a class. It refers to the current instance (object) of the class.

### `super()` function
**Answer** The `super()` function is a built-in function that returns a temporary proxy object of a parent (superclass) or sibling class, allowing you to access and call methods defined in that parent class from within a subclass.

```
class Robot:
    def __init__(self, name):
        self.name = name
        print(f"Robot {self.name} initialized.")

    def say_hello(self):
        print(f"Hello, I am {self.name}.")

class CleaningRobot(Robot):
    def __init__(self, name, battery_life):
        # Use super() to call the Parent's __init__ method
        super().__init__(name)
        
        # Add a new attribute specific to CleaningRobot
        self.battery_life = battery_life
        print(f"CleaningRobot {self.name} is ready with {self.battery_life}% battery.")

# Creating an instance
my_bot = CleaningRobot("Wall-E", 95)
my_bot.say_hello()

```
`super()` isn't just for the `__init__` method. You can use it to extend the functionality of any method from the parent class.

```
class Bird:
    def fly(self):
        print("The bird flaps its wings.")

class Eagle(Bird):
    def fly(self):
        # Call the parent behavior first
        super().fly()
        # Add specific behavior
        print("The eagle soars high in the clouds!")

hawk = Eagle()
hawk.fly()
```

### What are "Magic Methods" in Python, and why are they used?
**Answer** Magic methods are special methods with fixed names that begin and end with double underscores (e.g., __init__, __str__).
- They are invoked internally by the Python interpreter to perform specific operations.
- They are used to implement operator overloading and to allow custom objects to interact seamlessly with Python's built-in syntax (like using + on a custom class or checking length with len()).

### What is the difference between `__init__` and `__new__`?
**Answer**
`__new__`: This is the actual constructor. It is a static method that creates and returns a new instance of the class. It is rarely overridden unless you are working with immutable types (like int or str) or meta-programming.

`__init__`: This is the initializer. It is called after the instance has been created by `__new__`. It populates the object with attributes.

Think of `__new__` as building the house and `__init__` as decorating the interior.

### How do `__str__` and `__repr__` differ?
**Answer** Both return a string representation of an object, but they serve different purposes:

`__str__`: Aimed at the end-user. It should be readable and "informal." (Triggered by print() or str()).

`__repr__`: Aimed at developers. It should be unambiguous and, if possible, look like the Python code used to create the object. (Triggered by typing the variable in a shell or repr()).

Note: If `__str__` is not defined, Python will fall back to using `__repr__`.
```
class Car:
    def __init__(self, brand, model, year):
        self.brand = brand
        self.model = model
        self.year = year

    def __str__(self):
        # What the user sees
        return f"{self.year} {self.brand} {self.model}"

    def __repr__(self):
        # What the developer needs to recreate the object
        return f"Car(brand='{self.brand}', model='{self.model}', year={self.year})"

# Let's create an object
my_car = Car("Toyota", "Corolla", 2022)

# This calls __str__
print(str(my_car))   # Output: 2022 Toyota Corolla
print(my_car)        # Output: 2022 Toyota Corolla (print calls str by default)

# This calls __repr__
print(repr(my_car))  # Output: Car(brand='Toyota', model='Corolla', year=2022)
```

### Which magic method would you use to make an object "callable" like a function?
**Answer** You use the `__call__` method. When this method is defined in a class, you can instantiate the class and then "call" that instance using parentheses.

```
class Greeter:
    def __call__(self, name):
        return f"Hello, {name}!"

greet = Greeter()
print(greet("Alice")) # Works like a function
```

### How do you implement the "Context Manager" protocol (the with statement) in a class?
**Answer** To make a class compatible with the with statement, you must implement:

`__enter__`: Executed when the execution flow enters the context. It usually returns the resource (like a file or a database connection).

`__exit__`: Executed when the flow leaves the context. It handles teardown logic, such as closing a file or releasing a lock, even if an exception occurred.

```
class FileManager:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode
        self.file = None

    def __enter__(self):
        print(f"--- Opening {self.filename} ---")
        self.file = open(self.filename, self.mode)
        return self.file  # This is what 'f' becomes in 'with ... as f'

    def __exit__(self, exc_type, exc_val, exc_tb):
        print(f"--- Closing {self.filename} ---")
        if self.file:
            self.file.close()
        # If an error occurred, exc_type will not be None
        if exc_type:
            print(f"An error occurred: {exc_val}")
        return True # Suppresses the exception (optional)

# Using the context manager
with FileManager("test.txt", "w") as f:
    f.write("Hello, World!")
    print("Writing data...")

print("Done!")
```

#### The `__exit__` Parameters

The three arguments in `__exit__` are used to handle errors:

-   `exc_type`: The class of the exception (e.g., `ZeroDivisionError`).
-   `exc_val`: The instance of the exception (the error message).
-   `exc_tb`: The traceback object (the line numbers where it failed).

If `__exit__` returns `True`, Python swallows the error. If it returns `False` (the default), the error continues to bubble up.

### What are the magic methods for operator overloading?
**Answer** Python maps standard operators to specific dunder methods. Common examples include:

Arithmetic:
`__add__` (+)
`__sub__` (-)
`__mul__` (*)
`__truediv__` (/)

Comparison:
`__eq__` (==)
`__lt__` (<)
`__gt__` (>)
`__ne__` (!=)

Augmented Assignment:
`__iadd__` (+=)
`__isub__` (-=)

### How can you make a custom object behave like a container (list or dictionary)?
**Answer** To implement container-like behavior, you use the following:

`__len__`: To support len(obj).

`__getitem__`: To support indexing/key lookup obj[key].

`__setitem__`: To support assignment obj[key] = value.

`__delitem__`: To support deleting items del obj[key].

`__iter__`: To support iteration (e.g., in a for loop).

### What is the `__slots__` magic attribute?
**Answer** While technically an attribute rather than a method, `__slots__` tells Python not to use a dynamic dictionary (`__dict__`) to store instance attributes. Instead, it allocates a fixed amount of space for a static set of attributes. This significantly reduces memory usage and slightly speeds up attribute access for classes with many instances. The class is called The Slotted Class.

```
class SlottedPlayer:
    # Python will now reserve space ONLY for these two variables
    __slots__ = ['name', 'level']

    def __init__(self, name, level):
        self.name = name
        self.level = level

p2 = SlottedPlayer("Bob", 5)

# Attempting to add a new attribute will crash!
try:
    p2.health = 100
except AttributeError as e:
    print(f"Error: {e}") 
    # Output: 'SlottedPlayer' object has no attribute 'health'
```

### What's the Difference Between `is` and `==`?
**Answer**
- `==` compares **values**
- `is` compares **identity** (same object in memory)

```
a = [1, 2, 3]
b = [1, 2, 3]
print(a == b)  # True (same values)
print(a is b)  # False (different objects)
```

### What's the Difference Between Class and Static Methods?
**Answer**
The primary difference lies in bound data: a class method is bound to the class itself, while a static method is not bound to the class or the instance.
- **Instance method**: `def method(self)` - It has access to instance data
- **Class method**: `@classmethod def method(cls)` - It has access to the class's state and can modify it, which affects all instances of that class.
- **Static method**: `@staticmethod def method()` - It behaves like a plain function that lives inside the class's namespace. It is used when a function is logically related to the class but doesn't need to access any internal class data.

### Explain Memory Management in Python
**Answer** It is an automated process that handles how memory is allocated for objects and reclaimed when they are no longer needed.

Python uses a private heap space and two primary mechanisms: Reference Counting and Generative Garbage Collection.

- Objects have reference count; when it reaches 0, memory is freed
- Garbage collector handles circular references using generational algorithm
- Objects divided into 3 generations; younger objects checked more frequently

### Mutable vs Immutable in Python
**Answer** 
Mutable: list, dict, set, bytearray, User-defined classes
Immutable: int, float, complex, string, tuple, frozenset, bytes

### Properties and `@property` decorator
**Answer**
The `@property` decorator is a built-in way to implement getters and setters. It allows you to transform a method into a read-only attribute or add validation logic when a value is assigned.
The `@property` decorator is used to define "managed attributes." It allows you to define a method that can be accessed like a standard instance variable (without parentheses).

It is primarily used for:
- Encapsulation: Protecting data by adding validation logic.
- Computed Attributes: Calculating a value on the fly rather than storing it.
- Backwards Compatibility: Changing an attribute's implementation without changing the public API of the class.

Example ------
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        """The Getter"""
        return self._radius

    @radius.setter
    def radius(self, value):
        """The Setter with validation"""
        if value < 0:
            raise ValueError("Radius cannot be negative")
        self._radius = value

    @radius.deleter
    def radius(self):
        """The Deleter"""
        del self._radius

---------------

### What is the difference between a Property and a regular Attribute?
**Answer**
Attributes: Store data directly in the object's __dict__. Accessing them is a simple memory lookup.

Properties: Act as "virtual" attributes. When you access a property, Python executes a specific method. This allows for logic (like logging or range checking) to occur automatically during access or assignment.

Example -----

class User:
    def __init__(self, age):
        self._age = age # Actual data stored here

    @property
    def age(self):
        print("Someone is checking the age!")
        return self._age

    @age.setter
    def age(self, value):
        if value < 0:
            raise ValueError("Age cannot be negative!")
        self._age = value

---------------

### Why is using `@property` preferred over traditional `get_attr()` and `set_attr()` methods?
**Answer**
Python follows the principle that "Uniform Access" is better. In languages like Java, you must use getAge(). In Python, you start with a simple attribute person.age.

If you later need to add validation, you can turn it into a `@property` without changing the code that uses the class (i.e., the user still writes person.age = 30 instead of switching to person.set_age(30)). This prevents "breaking changes" in your codebase.

### Can a property be "Read-Only"?
**Answer**
Yes. If you define a method with the `@property` decorator but do not provide a corresponding `@property_name.setter`, the attribute becomes read-only. Any attempt to assign a value to it will raise an `AttributeError`.

### What are "Computed Properties"?
**Answer** These are properties that don't necessarily store a value in an instance variable but calculate it based on other attributes.

Example -----
class Square:
    def __init__(self, side):
        self.side = side

    @property
    def area(self):
        return self.side ** 2  # Computed on the fly

--------------

### How does the `@property` decorator relate to Descriptors?
**Answer**
The `@property` decorator is actually a convenient "shortcut" for creating a Data Descriptor. Under the hood, Python's property mechanism uses the descriptor protocol (__get__, __set__, and __delete__ methods) to intercept attribute access.

### What is the difference between an Iterable and an Iterator?
**Answer**
Iterable: An object capable of returning its members one at a time. It must implement the __iter__ method, which returns an iterator. Examples: list, str, tuple.

Iterator: An object representing a stream of data. it implements __next__, which returns the next item in the sequence, and __iter__, which returns the iterator object itself.

### What is a Generator, and how does it differ from a normal function?
**Answer**
A generator is a special type of iterator defined using a function. Instead of the return keyword, it uses yield. When a normal function is called, it executes from the first line to the return and terminates. When a generator is called, it returns a generator object without immediately executing. Each time next() is called on the generator, it runs until it hits yield, "pauses" its state, and returns the value.

### Why would you use a Generator instead of a List?
**Answer** The primary reason is Memory Efficiency.
A List stores all its elements in memory (eager evaluation).

A Generator produces items one at a time only when requested (lazy evaluation). If you are processing a file with 10 million lines, a list would likely crash your memory, while a generator would only hold one line at a time.

### What is the purpose of the yield keyword?
**Answer** yield is used to turn a function into a generator. It does two things:

It provides a value to the caller (like return).

It suspends the function's execution and saves all local variables and the instruction pointer. When next() is called again, the function resumes exactly where it left off.

### Why should you avoid using mutable default arguments in functions?
**Answer** In Python, default arguments are evaluated only once at the time the function is defined, not every time it is called. If you use a mutable object (like a list) as a default, that same list is shared across every call to the function.

Solution: Use None as the default and initialize the list inside the function.

Example -----

def add_item(item, target=[]):
    target.append(item)
    return target

print(add_item(1)) # [1]
print(add_item(2)) # [1, 2] - The list persisted!

--------------

### Are tuples always immutable?
**Answer** A tuple itself is immutable (you cannot add, remove, or replace its elements). However, if a tuple contains a mutable object (like a list), that list can still be modified. The tuple's identity remains the same because it still points to the same memory address of that list.

### What is the difference between a List and a Tuple?
**Answer**
Mutability: Lists are mutable (can be changed); tuples are immutable (cannot be changed after creation).

Performance: Tuples are slightly faster and consume less memory because their size is fixed.

Usage: Lists are generally used for collections of homogenous items (e.g., ['apple', 'banana']). Tuples are often used for heterogenous data structures (e.g., a database record like ('John', 25, 'Engineer')).

### How does a Python Dictionary work internally?
**Answer** Dictionaries are implemented as Hash Tables. When you add a key, Python applies a hash function to the key to calculate an index in an underlying array. This allows for O(1) (constant time) average complexity for lookups, insertions, and deletions.

### What are the main characteristics of a Set?
**Answer** 
- Uniqueness: Sets automatically remove duplicate elements.
- Unordered: Sets do not maintain the order of insertion.
- Hash-based: Like dictionaries, sets use hashing for O(1) membership testing (checking if an item exists using the in keyword).
- Mutable: You can add or remove items, but the items themselves must be hashable (immutable).

### What is the difference between `dict.get('key')` and `dict['key']`?
**Answer**
dict['key']: Raises a KeyError if the key does not exist.

dict.get('key'): Returns None (or a default value you provide) if the key is missing, making it safer for handling uncertain data.

### What is a Closure in Python?
**Answer** A closure is a function object that "remembers" values in its enclosing lexical scope even if that scope is no longer active.

For a closure to exist, three things must happen:
- There must be a nested function (a function inside a function).
- The nested function must refer to a variable defined in the enclosing function.
- The enclosing function must return the nested function.

Example ------

def counter():
    count = 0
    def increment():
        nonlocal count  # Allows modification of 'count'
        count += 1
        return count
    return increment

c = counter()
print(c()) # 1
print(c()) # 2

--------------

### What is the nonlocal keyword used for in a closure?
**Answer** By default, variables in the outer scope are read-only within the nested function. If you try to modify an outer variable, Python treats it as a new local variable. The nonlocal keyword allows the nested function to modify a variable in the outer (enclosing) scope.

### What is a Decorator?
**Answer** A decorator is a design pattern that allows you to add new functionality to an existing object (usually a function or class) without modifying its structure. In Python, decorators are usually implemented as closures that take a function as an argument and return a "wrapped" version of that function.

### How do you write a decorator that accepts arguments?
**Answer** To pass arguments to a decorator (e.g., @repeat(times=3)), you need three levels of nested functions:

- The outermost function accepts the decorator arguments.
- The middle function accepts the function to be decorated.
- The innermost function (the wrapper) handles the actual logic and arguments of the original function.

### Can you apply multiple decorators to a single function?
**Answer** Yes. This is called "chaining" or "stacking" decorators. They are applied from the bottom up (closest to the function first).

### What is the difference between an Instance Variable and a Class Variable?
**Answer**
Instance Variables: Are unique to each instance of a class. They are defined inside methods (usually __init__) using self.variable_name. Changing the variable in one instance does not affect others.

Class Variables: Are shared by all instances of a class. They are defined directly inside the class body (outside any methods). Changing a class variable affects every instance that hasn't overridden it.

### What happens if you try to modify a Class Variable through an Instance?
**Answer** This is a common "trick" question. If you use `instance.class_variable = value`, Python doesn't change the class variable. Instead, it creates a new instance variable with the same name that "shadows" (hides) the class variable for that specific instance. To change the actual class variable for everyone, you must use `ClassName.variable_name`.

### When should you use a Class Variable?
**Answer**: Class variables are best for:

- Constants: Values that should be the same for all objects (e.g., MAX_SPEED).
- Shared State: Tracking data across all instances, such as a counter for how many objects of that class have been created.
- Default Values: Providing a default that can be overridden by specific instances if needed.

### What is the Method Resolution Order (MRO) in relation to variables?
**Answer** When you access an attribute (e.g., obj.attr), Python follows a specific search order:

- Look in the Instance dictionary (obj.__dict__).
- If not found, look in the Class dictionary (Class.__dict__).
- If not found, look through the Parent Classes (following the MRO).
- If still not found, trigger __getattr__.

### How does the __dict__ attribute differ between a class and its instance?
**Answer**

- instance.__dict__ contains only the data specific to that object (instance variables).

- Class.__dict__ contains class variables, methods, and metadata (like docstrings). This separation is why class variables are memory-efficient; the data is stored once in the class object rather than being duplicated in every single instance.

### What is an Abstract Base Class (ABC)?
**Answer** An ABC is a class that cannot be instantiated on its own. Its primary purpose is to define a common interface (a set of methods and properties) that all its subclasses must follow. In Python, you create an ABC by inheriting from `abc.ABC` and using the `@abstractmethod` decorator.

### Why use ABCs instead of regular inheritance?
**Answer** Regular inheritance allows you to override methods, but it doesn't force you to. ABCs provide a way to enforce a contract. If a subclass fails to implement even one `@abstractmethod`, Python will raise a TypeError the moment you try to create an instance of that subclass.

Example ------

from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        pass

class Square(Shape):
    def __init__(self, side):
        self.side = side
    
    def area(self): # Must implement this!
        return self.side ** 2

---------------

### Can an Abstract Method have code in it?
**Answer** Yes! This is a common misconception. An abstract method can have a body. Subclasses are still required to override it, but they can call the parent's implementation using `super().method_name()`. This is useful for providing common setup logic.

Example -------



---------------

### What are "Virtual Subclasses" (the register method)?
**Answer** Python allows you to register a class as a "virtual subclass" of an ABC, even if it doesn't inherit from it. This is useful for third-party classes that follow the required interface but you cannot change their source code to add inheritance.

Shape.register(MyCustomObject)
# issubclass(MyCustomObject, Shape) will now return True
























































































































**3-7 Years (Intermediate)**

- [ ] Understand MRO and multiple inheritance
- [ ] Explain LEGB scope rules

**7+ Years (Senior/Lead)**
- [ ] Design complex class hierarchies
- [ ] Explain descriptors and metaclasses
- [ ] Optimize memory with `__slots__`
- [ ] Understand weak references
- [ ] Explain generational garbage collection
- [ ] Design using OOP patterns (Singleton, Factory)
- [ ] Mentor on Python best practices

---



## FINAL WEEK PREPARATION

**Day 2**:
- Light review only
- Prepare 3-4 anecdotes from your projects
- Research the company/role

**Interview Day**:
- Arrive early
- Take a deep breath
- Think before answering
- Ask clarifying questions if needed

---

## CODE CHALLENGE
--------- 1
Custom Range-like Iterator.
Implement a class called ReverseRange that mimics the behavior of the built-in range() but in reverse. It should not create a list of numbers in memory; instead, it should yield the numbers one by one to save memory.

Requirements:
- Initialize with a start and stop value.
- Implement the Iterator Protocol (__iter__ and __next__).
- Raise StopIteration when the sequence is exhausted.

--------- 2
The Fibonaaci Iterator
A more advanced challenge would be involving "Infinite Iterators" (like an iterator that generates Fibonacci numbers forever)

--------- 3
The "Timer" Decorator
A very common interview task is to write a decorator that measures the execution time of any function it decorates.

--------- 4
The Payment Processor
Question: Design an ABC called PaymentProcessor with an abstract method `process_payment(amount)`. Then, create a `CreditCardProcessor` subclass. What happens if you forget to define `process_payment` in the subclass?









