# 基于富文本编辑器的$关键字智能匹配功能

富文本编辑器，Multi-function Text Editor, 简称 MTE, 是一种可内嵌于浏览器，可以对文字、图片等进行编辑的，所见即所得的文本编辑器。

<!-- 目前，市面上大多富文本编辑器是依靠浏览器提供的contenteditable以及execCommand这两大原生的API实现的。
，wangeditor则是其中之一首先，将外部容器（一般为div）的contenteditable属性设置为true，该容器中就可以进行键盘、鼠标等操作了。另外，添加一个工具栏（toolbar），借助 execCommand 命令，实现对容器中内容的编辑功能，比如：文字加粗、斜体、下划线功能等。
但是，可以发现的是基于这种方案的富文本编辑器功能都比较基础且有限，对于指定关键字的触发无法实现，以及根据键盘输入的内容进行动态的变量匹配也未有实现，而基于此逻辑进一步实现的智能提示匹配更是功能更是没有。 -->

在此，建议先了解下 选区（Selection）和范围（Range）的基本概念 （可以参考这篇文章[利用 javascript 实现富文本编辑器](https://juejin.cn/post/6844903508446019597)）

## 前言

前段时间，接到一个需求，实现一个邮件的模板编辑功能，对于前端来说，就是用户可以在现有的富文本编辑器中插入变量，具体表现就是：

1. 在编辑器中通过 `$` 关键字触发，弹出一个类似 Select 选择框的弹窗，提供选择指定变量
2. 在选择变量之后，在编辑器中以 `${xx}` 这种形式保存，并且以不同颜色高亮区别

业务上的表现是这样的，在逻辑触发时，后端通过 `${xx}` 去筛选替换变量为对应的值，然后发送邮件到对应的用户。

## 功能实现

为了方便用户的使用，我们目前设计了这几个功能点：

1. 在不影响富文本编辑器的基本功能下，通过按键 `$` 触发选择框弹出
2. 选中的列表项，会以 `${xx}` 这种形式自动插入到编辑器的当前选区中
3. 插入到编辑器中的变量以 `${xx}` 这种形式保存，并且支持高亮显示
4. 支持 `${xx}` 中变量的动态模糊查询（例如：当你在 `${}` 中输入 `send` 时，列表中的 `sendTime`、`sendCount` 应该是匹配在列表中的 ）
5. 支持对 `${xx}` 的变量识别响应（例如：光标从其他位置移动到 `${}` 中时，会实时地响应匹配其中的变量）

## $ 关键字触发

如何监听 `$` 关键字，还是比较简单的，通过监听键盘事件就可以实现，在匹配到 `$` 时，触发对应逻辑即可，不过，这里重点其实是实现 `${}`。

> 变量需要以 `${xx}` 这种形式保存，而且前端也需要通过 `{}` 去隔离同一行内的其他变量，从而区分当前变量。

在检测到 `$` 字符时，自动把 `{}` 拼接上，并且把光标移动到 `${}` 之中。

这里我以公司使用的 wangeditor v4 版本为例，不过，本文所提到的功能实现不局限于某一固定框架，主要是基于原生实现。

### 页面代码基本结构

DOM结构

```html
<template>
  <div class="box">
    <!-- 编辑器 -->
    <div class="editor—wrapper">
      <div id="toolbar-container"><!-- 工具栏 --></div>
      <div id="text-container"><!-- 编辑器 --></div>
    </div>
    <!-- 弹窗 -->
    <div ref="tipRef" v-show="isShow" class="tip-wrap">我是弹窗</div>
  </div>
</template>
```

JS相关代码

```js
import { onMounted, ref, shallowRef } from 'vue'
import E from 'wangeditor'

// wangeditor编辑器实例
const editor = shallowRef()
// 编辑器DOM
const editorBox = shallowRef()
// 弹窗DOM
const tipRef = shallowRef()
// 弹窗开关
const isShow = ref(false)

// Mounted
onMounted(() => {
  editorBox.value = document.getElementById('text-container')

  initEditor()
  initEvents()
})

function initEditor() {
  editor.value = new E('#toolbar-container', '#text-container')
  editor.value.create()
}

function initEvents() {
  // 点击非弹窗区域，关闭弹窗
  window.addEventListener('click', windowClickClose)
  // 这里需要使用 keyup 键盘事件（${}），keydown 事件在处理$关键字时会导致 {$} 位置错误
  window.addEventListener('keyup', onWindowKeyUp, true)
  // selection
  document.addEventListener('selectionchange', selectionchange)
  // 在弹窗开启时，阻止对应键盘操作影响编辑器
  document.addEventListener('keydown', onWindowKeyDown)
}
```

### 绑定keyup事件

```js
window.addEventListener('keyup', onWindowKeyUp, true)

// 使用防抖函数 debounce 避免重复插入
const onWindowKeyUp = debounce((ev) => {
  // console.log('key', ev.key)
  const key = ev.key

  if (key === '$') {
    // isShow.value=true
    const selection = document.getSelection()
    const range = selection.getRangeAt(0)
    const textNode = range.startContainer
    // 获取光标位置
    const rangeStartOffeset = range.startOffset
    // 在 $ 后面插入 {}
    textNode.insertData(rangeStartOffeset, '{}') 
    // 移动光标位置
    range.setStart(textNode, rangeStartOffeset + 1)
    range.collapse(true)
    selection.removeAllRanges()
    // 插入新的光标对象
    selection.addRange(range)
  }
}, 10)
```

目前该函数可以实现在按下 `$` 按键后，输入 `${}` 并且光标在花括号之中。

按照设想，这时应该再自动触发 `isShow.value=true`，开启弹窗的逻辑，但是，因为在后面的实践过程中，发现这种方式并不好，涉及到用户体验的问题，相应的逻辑请往下看。

## 选中项自动插入到编辑器中

选中的列表项，会以 `${xx}` 这种形式自动插入到编辑器的当前选区中

简单画一下列表结构：

```html
<!-- 弹窗 -->
<div ref="tipRef" v-show="isShow" class="tip-wrap">
  <ol>
    <li 
      v-for="item in listData" 
      :key="item.value" 
      :data-value="item.value"
    >
      {{ `${item.label}(${item.value})` }}
    </li>
  </ol>
</div>
```

为了优化性能，我们把点击事件托管在外层DOM上

```js
// 列表点击事件
if (tipRef.value) {
  tipRef.value.addEventListener('click', onOlClick, true)
}

function onOlClick(ev) {
  const { value } = ev.target.dataset
  console.log('value', value)
}
```

点击选择这类基本功能不做过多阐述，这里主要说明下如何以 `${xx}` 这种形式自动插入到编辑器的当前选区中

### document.execCommand 方法

当一个div元素被添加`contenteditable`属性且为true时，表示元素可被用户编辑（富文本编辑器的基本逻辑）

该命令是浏览器提供的API，用于配合被`contenteditable`的元素，该方法允许运行命令来操纵可编辑内容区域的元素。

> document.execCommand 该方法虽然主流浏览器都支持，但现在官方已不再推荐使用，相关分析可以看这篇文章[🎉 富文本编辑器初探](https://juejin.cn/post/6955335319566680077)
>
> 分析源码可知，wangeditor v4版本及以下是基于该属性方法实现的，而 v5.x 版本内核已经完全重构，现在是基于 slate.js 开发的

这里我们还是以这个方案编写用例进行分析，主要讲解下基本思路：

```js
function insertField(val) {
  // 删除光标所在位置的字符，这里删除 } 字符
  editor.value.cmd.do('forwardDelete')
  // 获取位置
  let startOffset = document.getSelection().focusOffset
  // 当 ${} 中存在已输入的匹配字符时，需要计算对应字符长度
  if (keyword_str.value) {
    startOffset = document.getSelection().focusOffset - keyword_str.value.length
  }

  startOffset -= 2 // 计算 ${ 2个字符
  startOffset = startOffset < 0 ? 0 : startOffset // 不可为 负数

  // 重新设置选区位置
  document
    .getSelection()
    .getRangeAt(0)
    .setStart(
      document.getSelection().focusNode,
      startOffset
    )

  // 插入内容
  editor.value.cmd.do('insertHTML', `<span style="color: blue;">\${${val}}</span>`)
  // 插入空白占位符
  editor.value.selection.createEmptyRange()
}
```

在选中列表中的某一项时，传入 `value`，调用 `insertField` 方法

把`value`通过 `${}` 进行包裹，得到 `${value}` 这种形式，同时以 `span` 标签进行包裹，以实现字符隔离与颜色高亮

这里的重点是如何恰好替换编辑器中的 ${} 文本，通过 `Range` 对象的 `setStart` 方法，可以重新设置起始位置，

这里计算 `${}` 的前2个字符，得到 -2 的结果，同时删除末尾的 `}` 字符，然后再把结果重新插入到编辑器中即可

`editor.cmd.do` 是 wangeditor 基于 `document.execCommand` 方法做的一层封装，我们可以直接使用，相关代码见 [wangEditor/blob/v4.7.13/src/editor/command.ts](https://github1s.com/wangeditor-team/wangEditor/blob/v4.7.13/src/editor/command.ts)

最后一行的插入空白占位符 `createEmptyRange` 代码简要说明下：

1. 富文本编辑器默认是通过 `<br>` 充当一个空占位符的，初始结构一般是 `<p><br></p>`，F12查看DOM结构可知（<br>标签用来占位，有内容输入后会自动删除）
2. 在可编辑状态下，回车换行产生的新结构会默认拷贝之前的内容，包扩节点，类名等各种内容

这里插入空白占位符的主要作用：

1. 把光标移出到包裹变量的 span 标签外面
2. 编辑器内回车换行不会复制 span 标签影响内容编辑

> wangeditor 中对于 `createEmptyRange` 的实现，主要逻辑为 `editor.cmd.do('insertHTML', '&#8203;')`

## 支持变量的动态模糊查询

可选列表，当存在很多条数据时，再通过鼠标一个个找就很不友好了，因此，模糊查询也是很有必要的，那么，在这里，我们该如何设计和实现呢？

首先，需要确定的是如何锁定输入区域，有这么几种情况：

1. 在编辑器文本末尾输入
2. 在编辑器任意一行的任意位置输入

这里我们排除了编辑器的 `onChange` 方法，对于文本内任意位置插入编辑难以确定输入，同时也为了在鼠标或者键盘方向键移动到变量区域时能够有更好的交互响应，我选择了 `selectionchange` 事件，监听光标选区。

```js
document.removeEventListener('selectionchange', selectionchange)

function selectionchange() {
  const selection = document.getSelection()
  const range = selection.getRangeAt(0)
  // console.log('range', range)
}
```

通过 selection 对象获得的 range 对象才是我们操作光标的重点。Range表示包含节点和部分文本节点的文档片段。

### 1. 确定光标位置，查找 ${}

Range对象的 startContainer , endContainer, commonAncestorContainer都为 #text 文本节点，对于我们的场景来说并无明显区别，这里我们只查找 ${} 包裹的情况，不考虑跨多个文本节点的复杂场景，只围绕当前单个文本节点处理即可。

startOffset 和 endOffset 在非托选的情况下，起始和终点值是一样的，对于拖选情况来说，我们的处理方式也没什么区别（拖选的场景基本也可以不用考虑）

那么有一点需要确定的就是：**我当前的光标是否处于 ${} 之中？**

换个角度思考，我们看下光标处于 ${} 之中是怎样的：

1. 光标的左边应该是 ${
2. 光标的右边应该是 }

> 在此之前，我们在触发 $ 时，自动拼接了 {} ，从而完成了 ${} 这样的组合，这样可以很好的保证用户输入的一致性，方便我们后面的识别处理。

```js
// 获取当前文本节点的 value
const value = range.commonAncestorContainer.data
// 只有在当前文本包含有 $ 时，才考虑进一步处理
if (value && value.indexOf('$') > -1) {
    // 1. 查找 ${}
    // startOffset、endOffset 一般情况下都是相同的，没有拖选时，起终位置一致
    let start = range.startOffset // 
    let end = range.endOffset
    // 向左查找最近的 ${ 
    while (start > -1) {
      if (value[start - 1] === '$' && value[start] === '{') {
        break
      }
      start--
    }
    // 向右查找最近的 }
    while (end < value.length + 1) {
      if (value[end] === '}') {
        break
      }
      end++
    }

    // 2. 确定光标位置是否在 ${} 内
    if ((range.startOffset > start && start >= 0) && (range.endOffset <= end && end <= value.length)) {
      // ...
    }
}
```

通过上述代码，可以确定符合条件的 start 与 end 的值，即 ${ 和 } 的起点和终点。

那么下一步，我们取到里面的值，进行过滤查找即可

### 2. 查找关键字

比如我输入 `${start}`，那么取值就是 start，

且列表数据中包含 startTime、startGame、endTime、endGame 这些值，那么过滤查询符合匹配的应该有 startTime、startGame

`const currentStr = value.slice(start + 1, end)`

先得到 currentStr 值，并缓存到 `keyword_str.value = currentStr`，用于插入替换文本时计算位置使用

```js
if ((range.startOffset > start && start >= 0) && (range.endOffset <= end && end <= value.length)) {
  const currentStr = value.slice(start + 1, end)
  // 嵌套的不处理
  if (currentStr.indexOf('$') > -1 || currentStr.indexOf('{') > -1 || currentStr.indexOf('}') > -1) return

  // 3. 处理关键字匹配
  // 缓存关键字，用于插入替换文本时计算位置
  keyword_str.value = currentStr

  // 过滤匹配模糊查找
  const options = listData.value.filter(n => n.value.toLowerCase().includes(currentStr.toLowerCase()))
  tipOptions.value = clonedeep(options)

  // 打开弹窗
  // isShow.value = true
} else {
  // 不符合条件时，关闭弹窗，清空数据
  isShow.value = false
  tipOptions.value = []
}
```

### 动态计算弹窗位置

我想要的效果是类似 VsCode 代码提示那种，当我在输入时，弹窗的位置应该是始终跟随我光标的位置下面的

这里我们使用绝对定位absolute，只需要考虑 top 值即可，使用 `getBoundingClientRect` 方法快速实现

```js
// 4. 弹窗高度处理

// 弹窗 top 值（为了弹窗始终跟随光标所在位置）
const boxTop = ref(42)

const editorDomRect = editorBox.value.getBoundingClientRect() // 富文本容器
const textNodeRect = selection.focusNode.parentNode.getBoundingClientRect() // 文本节点
const top = textNodeRect.top - editorDomRect.top // 计算差值
// 1）需要考虑有的一串文本很长，可能横框几行，但是属于一个文本节点，因此，需要考虑 textNodeRect.height 
// 2）考虑富文本工具栏的高度 42
boxTop.value = top + textNodeRect.height + 42
```

一切就绪，控制弹窗的开启和关闭

是的，上面我们删除了在键盘事件输入 $ 时触发开启弹窗的逻辑，放在了这里，想必看到这里，你应该理解了在 selectiononchange 中进行处理的好处，尤其是针对我们的这个场景来说。

```js
if ((range.startOffset > start && start >= 0) && (range.endOffset <= end && end <= value.length)) {
  // ...

  // 打开弹窗
  isShow.value = true
} else {
  // 不符合条件时，关闭弹窗，清空数据
  isShow.value = false
  tipOptions.value = []
}
```

完整的代码，地址已放在下面了，感兴趣的可以进一步查看~

## 空白符的隔离优化处理

其实，到这一步，我们基本实现了相关功能。过，对于目前这种基于原生接口实现的富文本编辑器方案，还是存在一个非常影响体验的地方需要处理优化。

在上面的代码中，我们实现了插入逻辑，与此同时我们需要同步创建一个空白节点，并使光标向后移动一位，为的是在富文本的逻辑中，插入文本后不影响后面的输入。

虽然空白节点不可见，但是在我们通过鼠标点击的时候，会出现这么一种情况，在点击之后，光标的位置实际上是移动到了变量的文本节点内了的，例如：

`${startTime}abc` 这段文本

![1](https://cdn.jsdelivr.net/gh/JS-banana/images/vuepress/mte-hint-1.jpg)

在DOM上的结构实际上是这样的

![2](https://cdn.jsdelivr.net/gh/JS-banana/images/vuepress/mte-hint-2.jpg)

当我在 `}a` 这两个字符中间的位置点击一次时，光标会移动到这里，那么你觉得我接下来输入内容后，它会在哪个节点中呢？

答案是这样的：

1. 针对用例这样的内容，结果并不绝对，可能和我点击的位置有关系，有时在变量文本节点内，有时不在，看似正常，但是这样的结果非常不可靠，不是我们所能控制的
2. 对于 `${startTime}` 这样的文本，位于文本末尾时，基本上是非常确定的，在点击后，会跑到 span 标签中——我们的变量文本节点内。

那么我所希望的是，存在 `${startTime}` 这种插入后的变量文本标签时，当我点击 `}` 后面的位置，那么它就应该是处于变量文本节点外。

分析下情况：

1. 首先要判断光标位置是否在变量后面和空占位符前面
2. 变量文本节点处在末尾
3. 变量文本节点处在行中
4. 键盘操作的联动处理，如 删除键

同样的，这里的逻辑也是放在 `selectionchange` 事件中处理，相关逻辑抽离到 `filterEmptyText` 函数中

```js
function selectionchange() {
  const selection = document.getSelection()
  const range = selection.getRangeAt(0)
  // console.log('range', range)
  filterEmptyText(selection, range)
}
```

这里需要注意的是：在键盘处于操作的时候，恰当地控制函数的执行。

- 因为编辑区光标的变动会直接触发该函数的执行，需要避免产生逻辑冲突
- 删除键也需要单独处理，它会直接影响光标的位置是否在变量文本节点末尾
- 为了更好的控制变量节点，这里考虑视该节点为一个整体进行操作，尤其是在删除时

```js
function filterEmptyText(selection, range) {
  // const selection = document.getSelection()
  // const range = selection.getRangeAt(0)

  // 获取当前文本节点的父节点
  const parentNode = range.commonAncestorContainer.parentNode
  // 是否处在变量文本节点的末尾
  const isLast = range.commonAncestorContainer.length === range.endOffset
  // 当键盘按键处于操作中时，相关逻辑不处理
  if (parentNode && isLast && !keywordObj.isDoing) {
    const isVarSpan = parentNode.nodeName === 'SPAN' && /\$\{(.+?)\}/.test(parentNode.innerHTML)
    const isNextSibing = parentNode.nextSibling && parentNode.nextSibling.nodeName === 'SPAN'
    // 控制光标和空白符的关系
    if (isVarSpan && isNextSibing) {
      // 1. 空白占位符存在，向后移动一位
      console.log('符合1')
      const myRange = document.createRange()
      myRange.selectNodeContents(parentNode.nextSibling)
      myRange.collapse(true)
      selection.removeAllRanges()
      selection.addRange(myRange)
    } else if (isVarSpan && !isNextSibing) {
      // 2. 空白占位符不存在，创建
      console.log('符合2')
      if (keywordObj.isDeleteKey) {
        // 删除逻辑，直接删除整个变量，通过 isDeleteKey 避免影响光标位置
        parentNode.parentNode.removeChild(parentNode)
      } else {
        // 插入空白占位符
        editor.value.selection.createEmptyRange()
      }
    }
  }
}
```

## 总结

## demo地址

## 资料

- [选择（Selection）和范围（Range）](https://zh.javascript.info/selection-range)
- [JavaScript中对光标和选区的操作](https://segmentfault.com/a/1190000040211043)
- [document.execCommand](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/execCommand)
- [web dom api中的Selection和Range](https://www.cnblogs.com/kidsitcn/p/11628822.html)
- [https://www.wangeditor.com/v5/](https://www.wangeditor.com/v5/)
- [🎉 富文本编辑器初探](https://juejin.cn/post/6955335319566680077)
- [利用 javascript 实现富文本编辑器](https://juejin.cn/post/6844903508446019597)
