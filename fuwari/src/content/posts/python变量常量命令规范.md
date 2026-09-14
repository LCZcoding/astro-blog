---
title: python变量常量命令规范
published: 2026-09-14
description: '本文主要讲解了python的变量常量的命名规范和魔术方法与特殊变量的介绍'
image: ''
tags: [笔记, python]
category: '笔记'
draft: false 
lang: ''
---
- [x] 是否借助ai

# PEP 8 命名规范总表

| 类型 | 推荐风格 | 示例 |
|---|---|---|
| 变量 | 小写 + 下划线 | `user_name`, `total_count` |
| 常量 | 全大写 + 下划线 | `MAX_RETRY`, `DATABASE_URL` |
| 函数 | 小写 + 下划线 | `get_user()`, `calc_total()` |
| 方法 | 小写 + 下划线 | `save()`, `to_dict()` |
| 类 | 大驼峰 | `UserInfo`, `HttpClient` |
| 异常 | 大驼峰，常用 `Error` 结尾 | `ValueError`, `LoginError` |
| 模块 | 小写 + 下划线，短小 | `utils.py`, `data_loader.py` |
| 包 | 全小写，尽量不用下划线 | `mypackage`, `httptools` |
| 类型变量 | 大驼峰或单大写 | `T`, `AnyStr`, `UserType` |
| 私有成员 | 单下划线开头 | `_internal`, `_cache` |
| 名称改写 | 双下划线开头 | `__secret` |
| 魔术方法/特殊变量 | 双下划线开头和结尾 | `__init__`, `__name__` |

# 魔术方法与特殊变量

## 一、魔术方法

魔术方法：以 `__` 开头和结尾，由 Python 自动调用，不要自己直接调用，也不要自创。

### 常用分类

| 类别 | 方法 | 触发场景 |
|---|---|---|
| 生命周期 | `__new__` / `__init__` / `__del__` | 创建、初始化、销毁 |
| 字符串 | `__str__` / `__repr__` | `print()` / 调试表示 |
| 比较哈希 | `__eq__` / `__hash__` / `__bool__` | `==` / `hash()` / `bool()` |
| 容器迭代 | `__len__` / `__getitem__` / `__setitem__` / `__iter__` / `__next__` / `__contains__` | `len()` / 索引 / `for` / `in` |
| 属性访问 | `__getattr__` / `__setattr__` / `__delattr__` | 属性不存在 / 设置 / 删除 |
| 调用上下文 | `__call__` / `__enter__` / `__exit__` | `obj()` / `with` |
| 运算符 | `__add__` / `__radd__` / `__iadd__` | `+` / 右加 / `+=` |

### 示例

```python
class User:
    def __init__(self, name):
        self.name = name

    def __str__(self):
        return f"用户：{self.name}"

    def __repr__(self):
        return f"User(name={self.name!r})"

    def __eq__(self, other):
        return isinstance(other, User) and self.name == other.name

    def __hash__(self):
        return hash(self.name)
```

# 特殊变量

特殊变量由 Python 自动维护，通常不要随意覆盖。

| 变量 | 含义 |
|---|---|
| `__name__` | 模块名；直接运行时为 `"__main__"` |
| `__file__` | 当前模块文件路径 |
| `__doc__` | 文档字符串 |
| `__dict__` | 模块 / 类 / 实例的命名空间字典 |
| `__class__` | 实例所属类 |
| `__bases__` | 直接基类元组 |
| `__mro__` | 方法解析顺序 |
| `__all__` | `from module import *` 时导出的名字 |
| `__slots__` | 限制实例属性，节省内存 |
| `__version__` / `__author__` | 约定俗成的包信息 |

## 示例

```python
# main.py
def main():
    print("运行主程序")

if __name__ == "__main__":
    main()
```

```python
class User:
    __slots__ = ("name", "age")

    def __init__(self, name, age):
        self.name = name
        self.age = age
```

## 要点

- `if __name__ == "__main__":` 用于保护脚本入口。
- `__all__` 控制 `import *` 的导出内容。
- `__slots__` 会阻止动态添加属性。
- `__version__`、`__author__` 不是内置变量，只是社区约定。