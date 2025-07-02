# Python Typing

## @dataclass

1. **用途**：用于定义数据类，简化类的定义，自动实现`__init__`、`__repr__`、`__eq__`等方法。
2. **创建的对象**：实例对象，可以在运行时修改其属性。
3. **特性**：
    - 可以定义默认值、类型提示、方法等。
    - 支持继承和多态。
    - 可以进行运行时类型检查。
4. **示例**：

```python
from dataclasses import dataclass

@dataclass
class Person:
    name: str
    age: int = 0

person = Person(name='Alice', age=30)
print(person.name)  # 输出: Alice
person.age = 31
```

## TypedDict

1. **用途**：用于定义字典类型，可以指定键值对的类型。
2. **创建的对象**：字典对象，可以在运行时修改其键值对。
3. **特性**：
    - 只能定义键值对的类型。
    - 不支持继承和多态。
    - 类型检查主要在静态分析时进行 (例如使用 MyPy)。
    - 主要用于描述数据结构，而不是定义行为。
4. **示例**：

```python
from typing import TypedDict

class Movie(TypedDict):
    name: str
    year: int

movie: Movie = {'name': 'Blade Runner','year': 1982}
print(movie['name'])  # 输出: Blade Runner
# movie.year = 1949  # 报错 TypedDict 不支持属性访问方式修改
```
