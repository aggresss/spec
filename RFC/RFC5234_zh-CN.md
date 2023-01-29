# Augmented BNF for Syntax Specifications: ABNF

> 原文 [https://www.rfc-editor.org/rfc/rfc5234](https://www.rfc-editor.org/rfc/rfc5234)

## 摘要

互联网技术规范经常需要定义一种形式化的语法。近年来，一种改进的巴科斯范式(BNF)被称为增强 BNF (ABNF)，在许多互联网规范中得到广泛应用。现行规范文件 ABNF。它以合理的表示能力平衡了紧凑和简单性。标准 BNF 和 ABNF 之间的区别涉及命名规则、重复、可选性、顺序独立性和值范围。该规范还提供了一些其他的规则定义和编码，用于一些互联网规范中常见类型的核心词法分析器。

## 1. Introduction

互联网技术规范通常需要定义一种形式化的语法，并且可以自由使用作者认为有用的任何符号。近年来，一种改进的巴科斯范式(BNF)被称为增强 BNF (ABNF)，在许多互联网规范中得到广泛应用。它以合理的表示能力平衡了紧凑和简单性。在阿帕网的早期，每个规范都包含自己的 ABNF 定义。这包括电子邮件规范，[RFC733] 和 [RFC822]，它们成为定义 ABNF 的常见引用。当前的文档将这些定义分开，以允许选择性引用。可以预见的是，它还提供了一些修改和增强。
标准 BNF 和 ABNF 之间的区别涉及命名规则、重复、可选性、顺序独立性和值范围。附录 B 提供了几种互联网规范所共有的核心词法分析器的规则定义和编码。它是为了方便使用而提供的，它与本文档主体中定义的元语言是分开的，与它的正式状态也是分开的。

## 2. Rule Definition

### 2.1. Rule Naming

规则的名称就是名称本身，即一个字符序列，以字母字符开头，后面是字母、数字和连字符(破折号)的组合。

> 注意：规则名称不区分大小写。

规则名称 <rulename>, <Rulename>, <RULENAME>, <rUlENamE> 表示同一规则。

不像原始的 BNF，尖括号("<"，">")是不需要的。但是，只要尖括号的存在便于识别规则名的使用，就可以在规则名周围使用尖括号。这通常局限于自由格式的规则名称引用，或者区分组合成一个字符串而不是由空白分隔的部分规则，如下面关于重复的讨论中所示。

### 2.2. Rule Form

规则由以下序列定义:

```
    name = elements crlf
```

其中 <name> 是规则名称，<elements> 是一个或多个规则名称或终端规范，<crlf> 是行结束符(carriage return followed by line feed)。等号将名称与规则的定义分开。根据本文档中定义的各种操作符进行组合（例如 alternative 和 repetition），这些元素形成一个或多个规则名称 与/或 值定义的序列
为了方便查看，规则定义是左对齐的。当一条规则需要多行时，接续行会缩进。左对齐和缩进是相对于 ABNF 规则的第一行，不需要匹配文档的左外边距。

### 2.3. Terminal Values

规则解析为一个终端值字符串，有时称为字符。在 ABNF 中，字符只是一个非负整数。在某些上下文中，将指定值到字符集(如 ASCII )的特定映射(编码)。
终端由一个或多个数字字符指定，并明确指出这些字符的基本解释。目前定义的基如下:

```
    b = binary
    d = decimal
    x = hexadecimal
```

由此:

```
    CR = %d13
    CR = %x0
```

分别指定 [US-ASCII] 的十进制和十六进制表示形式。

由这些值拼接而成的字符串使用一个句点(".")来表示该值中的字符之间的分隔。因此:

```
    CRLF = %d13.10
```

ABNF 允许直接指定用引号括起来的字面文本字符串。因此:

```
    command = "command string"
```

字面量文本字符串被解释为一组连接起来的可打印字符。

> 注意: ABNF 字符串不区分大小写，这些字符串的字符集是 US-ASCII。

因此:

```
    rulename = "abc"
```

和:

```
    rulename = "aBc"
```

将匹配 “abc”, “Abc”, “aBc”, “abC”, “ABc”, “aBC”, “AbC”, “ABC”。

要指定区分大小写的规则，请分别指定字符。
例如:

```
    rulename = %d97 %d98 %d99
```

或

```
    rulename = %d97.98.99
```

将只匹配只包含小写字符 abc 的字符串。

### 2.4. External Encodings

终端值字符的外部表示形式会根据存储或传输环境的约束条件而变化。因此，相同的基于 ABNF 的语法可能有多种外部编码，例如一个用于 7 位 US-ASCII 环境，另一个用于二进制八位环境，在使用 16 位 Unicode 时仍然是不同的编码。编码细节超出了ABNF的范围，尽管附录 B 提供了 7 位 US-ASCII 环境的定义，这在因特网上是很常见的。
通过将外部编码从语法中分离出来，可以为相同的语法使用不同的编码环境。

## 3. Operators

### 3.1. Concatenation: Rule1 Rule2

### 3.2. Alternatives: Rule1 / Rule2

### 3.3. Incremental Alternatives: Rule1 =/ Rule2

### 3.4. Value Range Alternatives: %c##-##

### 3.5. Sequence Group: (Rule1 Rule2)

### 3.6. Varriable Repetition: *Rule

### 3.7. Specific Repetition: nRule

### 3.8. Optional Sequence: \[RULE\]

### 3.9. Comment: ; Comment

### 3.10. Operator Precedence

## 4. ABNF Definition of ABNF

## 5. Security Considerations

## 6. References

### 6.1. Normative References

### 6.2. Informative References

## Appendix A. Acknowledgements

## Appendix B. Core ABNF of ABNF

### B.1. Core Rules

### B.2. Common Encoding
