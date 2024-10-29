
# 大语言模型自如生成图案的一种可能的方式

## 立意

大语言模型已经可以很好的生成 基于 UI 的界面，但到目前为止并不能很好的生成 图案。

这主要是因为 SVG 没有完成他本来应该完成的任务。

SVG 号称是一种声明式的图形描述语言，但是在涉及坐标和尺寸上并不是，而是命令式的，和 canvas 没有什么区别。

所以导致大语言模型无法合理的描述图形之间的关系，但是我们在 css 中解决过这个问题，只是解决方案并不能完全迁移到图案描述上。

比方说：[Prompt 制作方法：文字逻辑关系图](https://mp.weixin.qq.com/s/d0y3e5ZJYDNi0tD-QXNA4g)

```lisp
(defun 智能绘制连接线 (x1 y1 x2 y2 &optional 曲线程度)
  "智能绘制灰色虚线箭头，避免穿过色块"
  (let ((dx (- x2 x1))
        (dy (- y2 y1))
        (mid-x (/ (+ x1 x2) 2))
        (mid-y (/ (+ y1 y2) 2)))
    (if 曲线程度
(path d ,(format "M%d,%d Q%d,%d %d,%d" 
                          x1 y1 
                          (+ mid-x (* dx 曲线程度)) (+ mid-y (* dy 曲线程度))
                          x2 y2)
               stroke="#808080" stroke-width="2" stroke-dasharray="5,5"
               fill="none" marker-end="url(#arrowhead)")
      `(path d ,(format "M%d,%d L%d,%d" x1 y1 x2 y2)
             stroke="#808080" stroke-width="2" stroke-dasharray="5,5"
             marker-end="url(#arrowhead)"))))
```

    目标：弥补 SVG 在布局和坐标上的不足，让其具备完整的声明式语言特性。
    方法：通过引入约束求解引擎 Kiwi.js，实现 SVG 中坐标和尺寸的自动求解，让图形的布局与定位由声明式约束控制。
    用途：增强大语言模型对 SVG 的理解，使其能在声明式框架下生成复杂图形。

## 实现步骤

1. 定义结构化的输出格式

    将 SVG 的定义分成三大部分，分别用于定义组件（defs）、元素（elements）、和布局约束（constraints），以实现完全声明式的图形生成。
    结构如下：
        defs：存储可复用的图形组件，所有组件统一用 symbol 标签定义，内部包含相对 symbol 的坐标。
        elements：在主 SVG 内容中通过 <use> 标签引用 defs 中的组件，禁止直接使用 x, y, width, height 等属性，由约束负责定位。
        constraints：使用 Kiwi.js 描述布局和定位，包含相对位置、对齐方式、间距等布局规则，并在求解后将坐标填入 elements。

2. 利用 Kiwi.js 实现坐标和布局的声明式控制

    引入约束求解：使用 Kiwi.js 定义布局约束，将原本需要手动设置的坐标和尺寸转变为自动求解的布局规则。
    实现声明式布局：例如，通过 verticalSpacing、alignCenterHorizontal 等约束，实现垂直间距、水平对齐等常用布局规则，让布局自适应内容变化。
    更新元素坐标：在布局求解后，动态更新 elements 中的 x, y 等位置属性，确保最终图形布局符合预期。

3. 设计适合大语言模型理解和生成的格式

    标准化结构：设计 JSON 格式使其结构一致，方便大语言模型理解并生成结构化的图形描述。
    生成声明式 SVG：通过大语言模型输出结构化的 JSON 描述，自动生成完整的 SVG 图形，实现模型直接生成复杂布局的能力。
    布局控制：约束的独立管理使布局与内容分离，大语言模型无需直接处理坐标问题，而是根据约束生成图形，使图形生成过程更加直观。

项目带来的改进

    声明式控制：通过布局约束，SVG 实现了更彻底的声明式图形描述，形成一个新的、更高层的 SVG 生成方式。
    图形生成自动化：大语言模型能够直接生成结构化的图形描述，具备生成复杂布局的能力。
    更强的灵活性：大幅提高 SVG 的布局灵活性，尤其在复杂图形、流程图、示意图等需要自适应的场景中，完全声明式的布局更易维护和扩展。

通过这个项目，你的 SVG 将不再局限于静态的绘图描述，而是具备了灵活的布局适应性和高效的生成能力，使其成为一个更强大的声明式图形语言。

通过 structure outputs 约束的生成数据结构：

```json
{
  "defs": [
    {
      "id": "nodeTemplate",
      "type": "symbol",
      "children": [
        { "type": "circle", "cx": 0, "cy": 0, "r": 20, "style": "fill: blue;" },
        { "type": "text", "x": 0, "y": 5, "content": "Node" }
      ]
    },
    {
      "id": "arrowTemplate",
      "type": "symbol",
      "children": [
        { "type": "path", "d": "M 0 0 L 10 10", "style": "stroke: black; fill: none;" }
      ]
    }
  ],
  "elements": [
    {
      "id": "A",
      "type": "use",
      "href": "#nodeTemplate",
      "label": "Start"
    },
    {
      "id": "B",
      "type": "use",
      "href": "#nodeTemplate",
      "label": "Step 1"
    },
    {
      "id": "C",
      "type": "use",
      "href": "#nodeTemplate",
      "label": "End"
    },
    {
      "id": "A_to_B",
      "type": "use",
      "href": "#arrowTemplate",
      "start": "A",
      "end": "B"
    }
  ],
  "constraints": [
    { "type": "verticalSpacing", "elements": ["A", "B", "C"], "spacing": 100 },
    { "type": "alignCenterHorizontal", "elements": ["A", "B", "C"] },
    { "type": "connect", "start": "A", "end": "B" },
    { "type": "connect", "start": "B", "end": "C" }
  ]
}
```

## 问题

现在还有一个问题，就是关于约束怎么写：

1. 如果约束完全使用 kiwi 的一对一约束来写，过于底层了，不能很好的表达需求本身。
2. 如果我创建一些内置的约束，我考虑怎么让这个约束集是合适的，不会丧失灵活性，也不会过于复杂

这是一个经典的抽象问题，在设计约束集时既要确保表达能力，又要避免引入不必要的复杂性。你可以考虑以下策略来平衡灵活性和简洁性：

1. 设计适中的内置约束集合

    从常用布局需求出发：以实际布局需求为基础设计约束，如垂直间距、对齐、分布等。每个约束都直接对应一个高层次的需求，这样可以减少手动管理低层次约束的繁琐性。
    内置约束示例：
        对齐约束：alignLeft, alignCenter, alignRight, alignTop, alignBottom
        间距约束：horizontalSpacing, verticalSpacing
        比例约束：equalWidth, equalHeight, aspectRatio
        连接约束：用于控制流程图中的箭头，自动连接节点 connect(start, end)

2. 将复杂的需求拆解为可复用的组合约束

    组合约束模板：为常见的布局需求创建组合模板，例如垂直对齐和均匀间距的组合。
    层次式嵌套：允许用户将内置约束组合成更复杂的自定义布局规则，便于高层次表达，但保持低层次约束的灵活性。

3. 提供低层次与高层次约束的双重接口

    高层接口：内置一组高层次的布局规则，供大多数情况使用，便于快速生成标准布局。
    底层接口：保留 Kiwi.js 的一对一约束接口，供有定制需求时直接使用。你可以允许用户在 JSON 中自定义单一变量约束，以便在必要时介入具体细节。

示例约束集结构

你的 JSON 约束可以分为两部分：高层内置约束和可选的底层自定义约束。例如：

```json
{
  "constraints": [
    { "type": "alignCenterHorizontal", "elements": ["A", "B", "C"] },
    { "type": "verticalSpacing", "elements": ["A", "B", "C"], "spacing": 100 },
    { "type": "aspectRatio", "element": "D", "ratio": 1.5 },
    { "type": "custom", "expression": "A.width == B.width * 2" }
  ]
}
```

在此结构中：

    高层内置约束通过 type 表达需求。
    低层自定义约束允许在 custom 内写表达式，供特殊需求时使用。

4. 提供灵活的“约束增强”机制

你还可以让用户定义自定义约束函数或扩展，以便动态扩展原有约束集。这种方式兼具简洁和灵活性，不会让基础约束集变得冗余。
总结

这种约束集设计不仅简化了高频需求的表达，还提供了灵活的低层次扩展选项，在使用者需要更细粒度控制时可以通过自定义实现。
