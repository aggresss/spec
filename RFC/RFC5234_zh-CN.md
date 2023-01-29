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

规则名称 `<rulename>`, `<Rulename>`, `<RULENAME>`, `<rUlENamE>` 表示同一规则。

不像原始的 BNF，尖括号("<"，">")是不需要的。但是，只要尖括号的存在便于识别规则名的使用，就可以在规则名周围使用尖括号。这通常局限于自由格式的规则名称引用，或者区分组合成一个字符串而不是由空白分隔的部分规则，如下面关于重复的讨论中所示。

### 2.2. Rule Form

规则由以下序列定义:

```
    name = elements crlf
```

其中 `<name>` 是规则名称，`<elements>` 是一个或多个规则名称或终端规范，`<crlf>` 是行结束符(carriage return followed by line feed)。等号将名称与规则的定义分开。根据本文档中定义的各种操作符进行组合（例如 alternative 和 repetition），这些元素形成一个或多个规则名称 与/或 值定义的序列
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

将匹配 "abc", "Abc", "aBc", "abC", "ABc", "aBC", "AbC", "ABC"。

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

### 3.1. Concatenation: `Rule1 Rule2`

通过列出一系列规则名称，规则可以定义一个简单的、有序的值字符串(即，连续字符的连接)。例如:

```
foo = %x61 ; a
bar = %x62 ; b
mumble = foo bar foo
```

所以，规则 `<mumble>` 匹配小写字符串 "aba"。

线性空白：拼接是 ABNF 解析模型的核心。连续字符(值)的字符串根据 ABNF 中定义的规则进行解析。对于互联网规范，允许线性空白符(空格和水平制表符)自由和隐式地穿插在主要结构周围已经有了一些历史，例如分隔特殊字符或原子字符串。

> 注意: 这个ABNF规范没有对线性空白空间的隐式规范作出规定。

任何希望允许分隔符或字符串段周围有线性空格的语法都必须明确指定它。在“核心”规则中提供这种空白通常很有用，这些空白可以在更高层次的规则中使用。“核心”规则可以构成词法分析器，也可以只是主要规则集的一部分。

### 3.2. Alternatives: `Rule1 / Rule2`

由正斜杠(“/”)分隔的元素是可选的。因此,

```
foo / bar
```

将接受 `<foo>` 或 `<bar>`。

> 注意: 包含字母字符的引号字符串是一种特殊形式，用于指定替代字符，并被解释为非终结符，表示包含字符的组合字符串的集合，该组合字符串按照指定的顺序，但可以使用任意大小写混合。

### 3.3. Incremental Alternatives: `Rule1 =/ Rule2`

有时，以片段的形式指定一组备选方案是很方便的。也就是说，初始规则可以匹配一个或多个备选项，后面的规则定义将添加到备选项集合中。这对于派生自同一父规则集的其他独立规范特别有用，例如经常与参数列表一起发生。ABNF 通过以下结构允许这种增量定义:

```
    oldrule =/ additional-alternatives
```

所以规则集:

```
    ruleset = alt1 / alt2
    ruleset =/ alt3
    ruleset =/ alt4 / alt5
```

等同于：

```
ruleset = alt1 / alt2 / alt3 / alt4 / alt5
```

### 3.4. Value Range Alternatives: `%c##-##`

可以使用破折号("-")来简洁地指定可选数值的范围。因此:

```
DIGIT = %x30-39
```

等价于:

```
    DIGIT = "0" / "1" / "2" / "3" / "4" / "5" / "6" /
        "7" / "8" / "9"
```

不能在同一个字符串中指定连接的数值和数值范围。数值可以使用点符号表示法进行连接，也可以使用虚线表示法指定一个取值范围。因此，要在行尾序列之间指定一个可打印字符，可以这样写:

```
    char-line = %x0D.0A %x20-7E %x0D.0A
```

### 3.5. Sequence Group: `(Rule1 Rule2)`

括号中的元素被视为单个元素，其内容是严格有序的。因此,

```
    elem (foo / bar) blat
```

匹配 (elem foo blat) 或 (elem bar blat)，并且

```
    elem foo / bar blat
```

匹配 (elem foo) 或 (bar blat)。

> 注意: 强烈建议使用分组表示法，而不是依赖于正确阅读直接替换，当替换由多个规则名称或字面量组成时。

因此，建议使用以下格式:

```
    (elem foo) / (bar blat)
```

这样可以避免普通读者的误解。

序列组表示法也用于自由文本中，以从普通文本中设置元素序列。

### 3.6. Varriable Repetition: `*Rule`

元素前面的运算符"*"表示重复。完整的表格是:

```
    <a>*<b>element
```

其中 `<a>` 和 `<b>` 是可选的十进制值，表示该元素至少出现 `<a>` 次，最多出现 `<b>` 次。
默认值为 0 和 无穷大，因此：

- `*<element>` 允许任何数字，包括 0;
- `1*<element>` 需要至少一个;
- `3*3<element>` 允许正好为 3;
- `1*2<element>` 允许一个或两个。

### 3.7. Specific Repetition: `nRule`

以下形式的规则:

```
    <n>element
```

等价于

```
    <n>*<n>element
```

也就是说，`<element>` 出现了 `<n>` 次。因此，`2DIGIT` 是一个2位数，而 `3ALPHA` 是一个由3个字母字符组成的字符串。

### 3.8. Optional Sequence: `[RULE]`

使用方括号将可选元素括起来：

```
    [foo bar]
```

等价于

```
 *1(foo bar)
```

### 3.9. Comment: `; Comment`

注释从分号开始，一直到行尾。这是在规范中同时包含有用注释的一种简单方法。

### 3.10. Operator Precedence

上述各种机制的优先级如下，从最高(绑定最严格)的顶部到最低(绑定最宽松)的底部:

```
    Rule name, prose-val, Terminal value
    Comment
    Value range
    Repetition
    Grouping, Optional
    Concatenation
    Alternative
```

替代操作符与拼接操作符随意混合使用，可能会让人感到困惑。

> 同样，建议使用分组操作符来创建显式的连接组。

## 4. ABNF Definition of ABNF

注意：

1. 这种语法要求相对严格的规则格式。因此，包含在规范中的规则集版本可能需要预处理，以确保它可以被ABNF解析器解释。
2. 此语法使用附录 B 中提供的规则。

```
rulelist = 1*( rule / (*c-wsp c-nl) )


```

## 5. Security Considerations

安全性被认为与本文档无关。

## 6. References

### 6.1. Normative References

- [US-ASCII] American National Standards Institute, "Coded Character Set -- 7-bit American Standard Code for Information Interchange", ANSI X3.4, 1986.
### 6.2. Informative References

- [RFC733] Crocker, D., Vittal, J., Pogran, K., and D. Henderson, "Standard for the format of ARPA network text messages", RFC 733, November 1977.
- [RFC822] Crocker, D., "Standard for the format of ARPA Internet text messages", STD 11, RFC 822, August 1982.

## Appendix A. Acknowledgements

ABNF 的语法最初在 [RFC733] 中指定。SRI International 的 Ken L. Harrenstien 负责将 BNF 重新编码为一个增强的 BNF，使表示更小，更容易理解。

这个最近的项目是为了剔除 [RFC822] 中被非电子邮件规范作者反复引用的部分，即增强 BNF 的描述。工作组没有简单、盲目地将现有的文本转换为单独的文件，而是选择仔细考虑现有规范和过去 15 年来提供的相关规范的不足和好处，并因此寻求加强。这使得该项目变得比最初设想的更加雄心勃勃。有趣的是，修改后的结果与最初的修改并没有太大的不同，不过一些决定，比如删除列表表示法，还是让人感到意外。

该规范的“分离”版本是 DRUMS 工作组的一部分，由 Jerome Abela、Harald Alvestrand、Robert Elz、Roger Fajman、Aviva Garrett、Tom Harsch、Dan Kohn、Bill McQuillan、Keith Moore、Chris Newman、Pete Resnick 和 Henning Schulzrinne 做出的重要贡献。

特别感谢 Julian Reschke 将该标准草案转换为 XML 源格式。

## Appendix B. Core ABNF of ABNF

本附录包含一些常用的基本规则。基本规则都是大写的。注意，这些规则只适用于 7 位 ASCII 编码的 ABNF，或 7 位 ASCII 超集的字符集。

### B.1. Core Rules

包含大写形式的基本规则，例如 SP, HTAB, CRLF, DIGIT, ALPHA 等：

```
    ALPHA = %x41-5A / %x61-7A ; A-Z / a-z

    BIT = "0" / "1"

    CHAR = %x01-7F
                ; any 7-bit US-ASCII
                ; excluding NUL

    CR = %x0D
                ; carriage return

    CRLF = CR LF
                ; Internet standard newline

    CTL = %x00-1F / %x7F
                ; controls

    DIGIT = %x30-39
                ; 0-9

    DQUOTE = %x22
                ; " (Double Quote)

    HEXDIG = DIGIT / "A" / "B" / "C" / "D" / "E" / "F"

    HTAB = %x09
                ; horizontal tab
    LF = %x0A
                ; linefeed

    LWSP = *(WSP / CRLF WSP)
                ; Use of this linear-white-space rule permits lines containing only white space
                ; that are no longer legal in mail headers and have caused interoperability problems
                ; in other contexts.
                ; Do not use when defining mail headers and use with caution in other contexts.

    OCTET = %x00-FF
                ; 8 bits of data

    SP = %x20

    VCHAR = %x21-7E
                ; visible (printing) characters

    WSP = SP / HTAB
                ; white space
```

### B.2. Common Encoding

在外部，数据表示为 "network virtual ASCII" (即 8 位字段中的 7 位 US-ASCII)，高位(第 8 位)设置为 0。字符串的值是按“网络字节顺序”排列的，左边表示高值的字节，首先通过网络发送。
