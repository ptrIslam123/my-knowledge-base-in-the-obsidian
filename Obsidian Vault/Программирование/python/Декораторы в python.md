## Декораторы классов

В Python можно динамически добавлять методы к классу (и к отдельным объектам). Это одна из ключевых особенностей языка. Вот основные способы:

## Добавление метода ко ВСЕМ экземплярам класса

```python
class MyClass:
    def __init__(self, name):
        self.name = name

# Определяем функцию, которая станет методом
def greet(self):
    return f"Hello, {self.name}!"

# Добавляем метод к классу
MyClass.greet = greet

# Теперь все экземпляры имеют этот метод
obj = MyClass("Alice")
print(obj.greet())  # Hello, Alice!
```

## Добавление метода к ОТДЕЛЬНОМУ объекту

```python
import types

class MyClass:
    def __init__(self, name):
        self.name = name

def greet(self):
    return f"Hello, {self.name}!"

obj1 = MyClass("Alice")
obj2 = MyClass("Bob")

# Добавляем метод только к obj1
obj1.greet = types.MethodType(greet, obj1)

print(obj1.greet())  # Hello, Alice!
# print(obj2.greet())  # AttributeError: 'MyClass' object has no attribute 'greet'
```

## 4. Использование `__getattr__` для динамической обработки

```python
class DynamicClass:
    def __init__(self):
        self.methods = {}
    
    def add_method(self, name, func):
        self.methods[name] = func
    
    def __getattr__(self, name):
        if name in self.methods:
            return self.methods[name].__get__(self, type(self))
        raise AttributeError(f"{name} not found")

def say_hello(self):
    return "Hello!"

obj = DynamicClass()
obj.add_method("hello", say_hello)
print(obj.hello())  # Hello!
```

## Декораторы для динамического добавления

Через декораторы добавляем новый метод классу
```python
def add_method(cls):
    def new_method(self):
        return "Added dynamically!"
    
    cls.new_method = new_method
    return cls

@add_method
class MyClass:
    pass

obj = MyClass()
print(obj.new_method())  # Added dynamically!
```

 или вручную, что семантически аналогично декораторам
```python
def add_method(cls):
    def new_method(self):
        return "Added dynamically!"
    
    cls.new_method = new_method
    return cls

MyClass = add_method(MyClass)
obj = MyClass()
print(obj.new_method())  # Added dynamically!
```

## Добавление статических и классовых методов

```python
class MyClass:
    pass

# Статический метод
@staticmethod
def static_method():
    return "Static"

# Классовый метод
@classmethod
def class_method(cls):
    return f"Class: {cls.__name__}"

MyClass.static_method = static_method
MyClass.class_method = class_method

print(MyClass.static_method())  # Static
print(MyClass.class_method())   # Class: MyClass
```

## 8. Использование `__init_subclass__` (Python 3.6+)

```python
class Base:
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        def new_method(self):
            return f"Added to {cls.__name__}"
        cls.new_method = new_method

class MyClass(Base):
    pass

obj = MyClass()
print(obj.new_method())  # Added to MyClass
```

## Важные нюансы:

1. **Привязка**: Обычные функции, добавленные к классу, автоматически становятся связанными методами (получают `self`).

2. **Слоты**: Если класс использует `__slots__`, динамическое добавление методов работает, но добавление атрибутов данных — нет.

3. **Дескрипторы**: Добавленные методы становятся дескрипторами, поэтому они правильно работают с `self`.

4. **Производительность**: Динамическое добавление методов работает медленнее, чем статически определённые методы, но в большинстве случаев разница незаметна.

## Практический пример: плагинная система

```python
class PluginSystem:
    plugins = {}
    
    @classmethod
    def register_plugin(cls, name):
        def decorator(func):
            cls.plugins[name] = func
            return func
        return decorator

# Динамически добавляем плагины
@PluginSystem.register_plugin("greet")
def greet_plugin(self):
    return "Hello from plugin!"

@PluginSystem.register_plugin("farewell")
def farewell_plugin(self):
    return "Goodbye from plugin!"

# Добавляем все плагины в класс
class MyApp:
    pass

for name, func in PluginSystem.plugins.items():
    setattr(MyApp, name, func)

app = MyApp()
print(app.greet())     # Hello from plugin!
print(app.farewell())  # Goodbye from plugin!
```

Как видите, Python предоставляет множество способов динамического добавления методов, что делает язык очень гибким и мощным для метапрограммирования.