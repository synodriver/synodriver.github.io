+++
date = '2026-05-04T22:39:16+08:00'
draft = false
title = 'PEP 7 – C 代码风格指南'
author = 'synodriver'
+++

# PEP 7 – C 代码风格指南

| 字段 | 内容 |
|------|------|
| 作者 | Guido van Rossum \<guido at python.org\>, Barry Warsaw \<barry at python.org\> |
| 状态 | 活跃（Active） |
| 类型 | 流程（Process） |
| 创建日期 | 2001-07-05 |

---

## 目录

- [引言](#引言)
- [C 标准](#c-标准)
- [通用 C 代码约定](#通用-c-代码约定)
- [代码排版](#代码排版)
- [命名约定](#命名约定)
- [文档字符串](#文档字符串)
- [版权](#版权)

## 引言

本文档为构成 Python C 实现的 C 代码提供了编码规范。请参阅描述 Python 代码风格指南的配套信息性 PEP。

请注意，规则是用来打破的。打破某条规则的两个正当理由：

1. 当应用该规则会使代码可读性降低时——即使对于习惯阅读遵循规则的代码的人来说也是如此。
2. 为了与周围同样违反该规则的代码保持一致（也许是出于历史原因）——尽管这也是清理他人遗留问题的好机会（以真正的 XP 风格）。

## C 标准

请遵循以下标准。对于相关标准中未涵盖的特性，请使用 CPython 专用的包装器（例如：`_Py_atomic_store_int32`、`Py_ALWAYS_INLINE`、`Py_ARITHMETIC_RIGHT_SHIFT`；公共头文件中的 `_Py_ALIGNED_DEF`）。在添加此类包装器时，尽量使其易于适配不支持的编译器。

- Python 3.11 及更新版本使用 C11（不包含可选特性）。公共 C API 应与 C99 和 C++ 兼容。

  （提醒阅读本文的各位用户：本 PEP 是一份风格指南；这些规则是可以被打破的。）

- Python 3.6 到 3.10 使用 C89 并包含若干选定的 C99 特性：
  - `<stdint.h>` 和 `<inttypes.h>` 中的标准整数类型。我们要求使用固定宽度的整数类型。
  - `static inline` 函数
  - 指定初始化器（designated initializers）（特别适合类型声明）
  - 交叉声明（intermingled declarations）
  - 布尔类型
  - C++ 风格的行注释

- Python 3.6 之前的版本使用 ANSI/ISO 标准 C（1989 年版本的标准）。这意味着，除其他事项外，所有声明必须位于代码块的顶部。

## 通用 C 代码约定

- 不要使用编译器特有的扩展，例如 GCC 或 MSVC 的扩展。例如，不要编写没有尾随反斜杠的多行字符串。
- 所有函数声明和定义必须使用完整的原型。也就是说，指定所有参数的类型，并使用 `(void)` 来声明没有参数的函数。
- 使用主流编译器（gcc、VC++ 以及其他几个）编译时不应产生任何警告。
- 在新代码中应优先使用 `static inline` 函数而非宏。

## 代码排版

- 使用 4 个空格缩进，完全不使用制表符。
- 任何行都不应超过 79 个字符。如果这条规则和前一条规则加在一起仍然没有给你足够的编码空间，那说明你的代码太复杂了——考虑使用子程序。
- 任何行都不应以空白字符结尾。如果你认为需要重要的尾部空白，请再想想——某个人的编辑器可能会例行删除它。
- 函数定义风格：函数名在第 1 列，最外层花括号在第 1 列，局部变量声明后空一行。

  ```c
  static int
  extra_ivars(PyTypeObject *type, PyTypeObject *base)
  {
      int t_size = PyType_BASICSIZE(type);
      int b_size = PyType_BASICSIZE(base);

      assert(t_size >= b_size); /* type smaller than base! */
      ...
      return 1;
  }
  ```

- 代码结构：在 `if`、`for` 等关键字和紧随其后的左括号之间加一个空格；括号内不加空格；所有地方都要求使用花括号，即使 C 允许省略它们，但不要在你未做其他修改的代码中添加花括号。所有新的 C 代码都必须使用花括号。花括号的格式如下所示：

  ```c
  if (mro != NULL) {
      ...
  }
  else {
      ...
  }
  ```

- `return` 语句不应使用多余的括号：

  ```c
  return(albatross); /* 错误 */
  ```

  应改为：

  ```c
  return albatross; /* 正确 */
  ```

- 函数和宏调用风格：`foo(a, b, c)`——左括号前无空格，括号内无空格，逗号前无空格，每个逗号后一个空格。
- 始终在赋值、布尔和比较运算符两侧加空格。在使用大量运算符的表达式中，在最外层（最低优先级）运算符两侧加空格。
- 断开长行：如果可能，在最外层参数列表的逗号后断行。始终适当缩进续行，例如：

  ```c
  PyErr_Format(PyExc_TypeError,
               "cannot create '%.100s' instances",
               type->tp_name);
  ```

- 在二元运算符处断开长表达式时，花括号的格式如下所示：

  ```c
  if (type->tp_dictoffset != 0
      && base->tp_dictoffset == 0
      && type->tp_dictoffset == b_size
      && (size_t)t_size == b_size + sizeof(PyObject *))
  {
      return 0; /* "原谅"仅添加 __dict__ 的情况 */
  }
  ```

  将运算符放在行末也是可以的，特别是为了与周围代码保持一致时。（参见 PEP 8 中的详细讨论。）

- 在多行宏中垂直对齐续行字符。
- 用作语句的宏应使用 `do { ... } while (0)` 宏惯用法，不带末尾分号。示例：

  ```c
  #define ADD_INT_MACRO(MOD, INT)                                   \
      do {                                                          \
          if (PyModule_AddIntConstant((MOD), (#INT), (INT)) < 0) {  \
              goto error;                                           \
          }                                                         \
      } while (0)

  // 像带分号的语句一样使用：
  ADD_INT_MACRO(m, SOME_CONSTANT);
  ```

- 在使用完毕后 `#undef` 文件局部宏。
- 在函数、结构体定义以及函数内部的主要段落之间加空行。
- 注释放在它们所描述的代码之前。
- 除非要成为已发布接口的一部分，所有函数和全局变量都应声明为 `static`。
- 对于外部函数和变量，我们总是在 "Include" 目录下的适当头文件中声明，使用 `PyAPI_FUNC()` 宏和 `PyAPI_DATA()` 宏，如下所示：

  ```c
  PyAPI_FUNC(PyObject *) PyObject_Repr(PyObject *);

  PyAPI_DATA(PyTypeObject) PySuper_Type;
  ```

## 命名约定

- 公共函数使用 `Py` 前缀；静态函数永远不要使用。`Py_` 前缀保留给全局服务例程，如 `Py_FatalError`；特定的例程组（例如特定对象类型 API）使用更长的前缀，例如字符串函数使用 `PyString_`。
- 公共函数和变量使用 MixedCase 加下划线，如：`PyObject_GetAttr`、`Py_BuildValue`、`PyExc_TypeError`。
- 偶尔某个"内部"函数需要对加载器可见；我们为此使用 `_Py` 前缀，例如：`_PyObject_Dump`。
- 宏应使用 MixedCase 前缀，然后使用大写，例如：`PyString_AS_STRING`、`Py_PRINT_RAW`。
- 宏参数应使用 ALL_CAPS 风格，以便与 C 变量和结构体成员区分开来。

## 文档字符串

- 使用 `PyDoc_STR()` 或 `PyDoc_STRVAR()` 宏来编写文档字符串，以支持在不带文档字符串的情况下构建 Python（`./configure --without-doc-strings`）。
- 每个函数文档字符串的第一行应该是"签名行"，简要概述参数和返回值。例如：

  ```c
  PyDoc_STRVAR(myfunction__doc__,
  "myfunction(name, value) -> bool\n\n\
  Determine whether name and value make a valid pair.");
  ```

  始终在签名行和描述文本之间包含一个空行。

  如果函数的返回值始终为 `None`（因为没有有意义的返回值），则不要包含返回类型的说明。

- 编写多行文档字符串时，务必始终使用反斜杠续行，如上例所示，或使用字符串字面量拼接：

  ```c
  PyDoc_STRVAR(myfunction__doc__,
  "myfunction(name, value) -> bool\n\n"
  "Determine whether name and value make a valid pair.");
  ```

  虽然某些 C 编译器接受不带任何一种方式的字符串字面量：

  ```c
  /* 错误——不要这样做！ */
  PyDoc_STRVAR(myfunction__doc__,
  "myfunction(name, value) -> bool\n\n
  Determine whether name and value make a valid pair.");
  ```

  但并非所有编译器都接受；已知 MSVC 编译器会对此报错。

## 版权

本文档已进入公共领域。
