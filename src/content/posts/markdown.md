---
title: Markdown 示例
published: 2023-10-01
description: 一份简洁的 Markdown 博客文章示例
tags: [Markdown, 博客]
category: Examples
draft: false
---

# 一级标题

段落之间用空行分隔。

第二段。*斜体*、**粗体** 和 `等宽代码`。无序列表的格式如下：

- 第一项
- 第二项
- 第三项

注意：不计星号的情况下，实际文本内容需从左侧第 4 列开始书写。

> 块引用的格式
> 示例如下。
>
> 块引用可跨越多段内容，
> 按需使用即可。

使用三个连字符表示全角破折号（——），两个连字符表示数值范围（例如：“内容全部在第 12--14 章”）。三个点号 ... 会被转换为省略号（……）。
支持 Unicode 字符显示。☺

## 二级标题

有序列表示例：

1. 第一项
2. 第二项
3. 第三项

再次注意：实际文本需从左侧第 4 列（4 个字符宽度）开始。以下是代码片段示例：

    # 再次强调...
    for i in 1 .. 10 { do-something(i) }

如你所见，这是缩进 4 个空格的效果。此外，也可使用分隔块替代缩进（更便于复制粘贴）：

```
define foobar() {
    print "Welcome to flavor country!";
}
```

你也可以为分隔块指定语言类型，让 Pandoc 自动进行语法高亮：

```python
import time
# 快，数到十！
for i in range(10):
    # （不用太快）
    time.sleep(0.5)
    print(i)
```

### 三级标题

嵌套列表示例：

1. First, get these ingredients:

    - carrots
    - celery
    - lentils

2. Boil some water.

3. Dump everything in the pot and follow
    this algorithm:

        find wooden spoon
        uncover pot
        stir
        cover pot
        balance wooden spoon precariously on pot handle
        wait 10 minutes
        goto first step (or shut off burner when done)

    Do not bump wooden spoon or it will fall.

再次强调：所有文本需在 4 个空格的缩进位置对齐（包括
上述第 3 项的最后一行）。

链接示例：[外部网站](http://foo.bar)、[本地文档](local-doc.html)、
[本文档内锚点](#二级标题)。脚注示例 [^1]。

[^1]: 脚注内容写在此处。

表格可按如下格式书写：

尺寸 | 材质 | 颜色
---- | ---- | ----
9    | 皮革 | 棕色
10   | 麻帆布 | 自然色
11   | 玻璃 | 透明

表：鞋子的尺寸与材质说明

（以上为表格标题）。Pandoc 同样支持多行表格：

| 关键词 | 文本内容 |
| ------ | -------- |
| 红色   | 日落、苹果，以及<br>其他红色或偏红的<br>事物。 |
| 绿色   | 树叶、草地、青蛙，<br>以及其他“生来不易”的<br>事物。 |

---

以下是一条水平线分割线。

---

定义列表示例：

苹果
: 适合制作苹果酱。
橙子
: 柑橘类水果！
西红柿
:   “西红柿”的正确拼写里没有字母 e（tomato 非 tomatoe）。

同样，定义文本需缩进 4 个空格。（在术语/定义对之间加空行可优化排版间距。）


行内数学公式写法：$\omega = d\phi / dt$。
块级数学公式需单独成行，并使用双美元符号包裹：

$$I = \int \rho R^{2} dV$$

$$
\begin{equation*}
\pi = 3.1415926535\;8979323846\;2643383279\;5028841971\;6939937510\;5820974944\;5923078164\;0628620899\;8628034825\;3421170679\;\ldots
\end{equation*}
$$

如需原样显示标点符号，可使用反斜杠转义：\`foo\`、\*bar\* 等。

# Markdown 扩展功能
## GitHub 仓库卡片
可添加关联 GitHub 仓库的动态卡片，页面加载时会自动从 GitHub API 拉取仓库信息：

::github{repo="Fabrizz/MMM-OnSpotify"}

创建仓库卡片的语法：`::github{repo="<仓库所有者>/<仓库名>"}`

```markdown
::github{repo="saicaca/fuwari"}
```

## 提示框（Admonitions）

支持以下类型的提示框：`note`（注释）、`tip`（提示）、`important`（重要）、`warning`（警告）、`caution`（注意）

:::note
需重点强调的信息，即使用户快速浏览也能注意到。
:::

:::tip
帮助用户提升使用体验的可选信息。
:::

:::important
用户完成操作必须知晓的关键信息。
:::

:::warning
存在潜在风险，需用户立即关注的核心内容。
:::

:::caution
提示某操作可能引发的负面后果。
:::

### 基础语法

```markdown
:::note
需重点强调的信息，即使用户快速浏览也能注意到。
:::

:::tip
帮助用户提升使用体验的可选信息。
:::
```

### 自定义标题

可自定义提示框的标题：

:::note[自定义标题]
这是一个带自定义标题的注释提示框。
:::

```markdown
:::note[自定义标题]
这是一个带自定义标题的注释提示框。
:::
```

### GitHub 语法兼容

> [!TIP]
> 也支持 [GitHub 原生语法](https://github.com/orgs/community/discussions/16925)。

```markdown
> [!NOTE]
> 兼容 GitHub 原生提示框语法。

> [!TIP]
> 兼容 GitHub 原生提示框语法。
```

### 剧透效果

可为文本添加剧透效果，剧透内容支持 **Markdown** 语法：

内容 :spoiler[被隐藏啦 **嘿嘿**]！

```markdown
内容 :spoiler[被隐藏啦 **嘿嘿**]！
```
