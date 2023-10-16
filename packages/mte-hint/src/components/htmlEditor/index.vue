<template>
  <div class="box">
    <!-- 编辑器外层容器 -->
    <div class="editor—wrapper">
      <div id="toolbar-container"><!-- 工具栏 --></div>
      <div ref="editorBox" id="text-container"><!-- 编辑器 --></div>
    </div>
    <!-- 弹窗 -->
    <div ref="tipRef" v-show="isShow" class="tip-wrap" :style="{ top: `${boxTop}px` }">
      <ol>
        <li v-for="item in tipOptions" :key="item.value" :data-value="item.value">
          {{ `${item.label}(${item.value})` }}
        </li>
      </ol>
    </div>
  </div>
</template>
<script setup>
import { onMounted, onUnmounted, reactive, ref, shallowRef } from 'vue'
import E from 'wangeditor'
import debounce from 'lodash.debounce'
import clonedeep from 'lodash.clonedeep'

// DOM 相关
const editor = shallowRef() // wangeditor编辑器实例
const editorBox = shallowRef() // 编辑器DOM
const tipRef = shallowRef() // 弹窗DOM

// 响应式数据相关
const isShow = ref(false) // 弹窗开关
const boxTop = ref(42) // 弹窗 top 值（为了弹窗始终跟随光标所在位置）

// 键盘按键相关
const keywordObj = reactive({
  currentStr: '', // ${} 中当前已输入的变量字符
  isDoing: false, // 按键是否为操作中
  isDeleteKey: false // 是否为删除键
})


// 列表数据源（一般是通过接口或者prop传入）
const listData = ref([
  { label: '开始时间', value: 'startTime' },
  { label: '结束时间', value: 'endTime' },
  { label: '开始游戏', value: 'startGame' },
  { label: '结束游戏', value: 'endGame' }
])
// 当前弹窗列表数据
const tipOptions = ref([])

// Mounted
onMounted(() => {
  // 缓存富文本DOM容器，后面使用
  // editorBox.value = document.getElementById('text-container')
  // 初始化时，默认赋值一次
  tipOptions.value = clonedeep(listData.value)

  // 初始化编辑器
  initEditor()
  // 初始化事件
  initEvents()
})

onUnmounted(() => {
  clearEvents()
})

function clearEvents() {
  window.removeEventListener('click', windowClickClose)
  window.removeEventListener('keyup', onWindowKeyUp, true)
  document.removeEventListener('selectionchange', selectionchange)
  document.removeEventListener('keydown', onWindowKeyDown)
  if (tipRef.value) {
    tipRef.value.removeEventListener('click', onOlClick, true)
  }
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
  // 列表点击事件
  if (tipRef.value) {
    tipRef.value.addEventListener('click', onOlClick, true)
  }
}

function initEditor() {
  editor.value = new E('#toolbar-container', '#text-container')
  editor.value.create()
}

function windowClickClose(ev) {
  if (tipRef.value && !tipRef.value.contains(ev.target)) {
    isShow.value = false
  }
}


const onWindowKeyUp = debounce((ev) => {
  console.log('key', ev.key)
  const key = ev.key

  keywordObj.isDoing = true
  keywordObj.isDeleteKey = false

  if (key === 'Backspace') {
    keywordObj.isDeleteKey = true
    // 删除按键时，需要手动调用一次 selectionchange，触发对应逻辑走一遍
    selectionchange()
    return
  }

  if (key === '$') {
    isShow.value = true
    const selection = document.getSelection()
    const range = selection.getRangeAt(0)
    const textNode = range.startContainer
    // 获取光标位置
    const rangeStartOffeset = range.startOffset
    textNode.insertData(rangeStartOffeset, '{}')
    // 移动光标位置
    range.setStart(textNode, rangeStartOffeset + 1)
    range.collapse(true)
    selection.removeAllRanges()
    // 插入新的光标对象
    selection.addRange(range)
  }
}, 10)


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

function selectionchange() {
  const selection = document.getSelection()
  // 非编辑区禁止触发
  if (!editorBox.value.contains(selection.anchorNode)) return

  const range = selection.getRangeAt(0)
  console.log('range', range)

  filterEmptyText(selection, range)

  // 确定光标位置
  // 过滤查找当前文本节点
  const value = range.commonAncestorContainer.data
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

    // 2. 确定光标位置 startOffset ~ endOffset，是否在 ${} 内
    if ((range.startOffset > start && start >= 0) && (range.endOffset <= end && end <= value.length)) {
      const currentStr = value.slice(start + 1, end)
      // 嵌套的不处理
      if (currentStr.indexOf('$') > -1 || currentStr.indexOf('{') > -1 || currentStr.indexOf('}') > -1) return

      // 3. 处理关键字匹配
      // 缓存关键字，用户插入替换文本时计算位置
      keywordObj.currentStr = currentStr

      // 过滤匹配模糊查找
      const options = listData.value.filter(n => n.value.toLowerCase().includes(currentStr.toLowerCase()))
      tipOptions.value = clonedeep(options)

      // 4. 弹窗高度处理
      const editorDomRect = editorBox.value.getBoundingClientRect() // 富文本容器
      const textNodeRect = selection.focusNode.parentNode.getBoundingClientRect() // 文本节点
      const top = textNodeRect.top - editorDomRect.top // 计算差值
      // 1）需要考虑有的一串文本很长，可能横框几行，但是属于一个文本节点，因此，需要考虑 textNodeRect.height 
      // 2）考虑富文本工具栏的高度 42
      console.log(textNodeRect.top, editorDomRect.top)
      boxTop.value = top + textNodeRect.height + 42

      // 打开弹窗
      isShow.value = true
    } else {
      // 不符合条件时，关闭弹窗，清空数据
      isShow.value = false
      tipOptions.value = []
    }
  }

  keywordObj.isDoing = false
}

function onWindowKeyDown(ev) {
  if (isShow.value && ['ArrowUp', 'ArrowDown', 'Enter'].includes(ev.key)) {
    ev.preventDefault();
    return false
  }
}

function onOlClick(ev) {
  const { value } = ev.target.dataset
  console.log('value', value)
  if (value) {
    insertField(value)
    isShow.value = false
  }
}

function insertField(val) {
  // 删除光标所在位置的字符，这里删除 } 字符
  editor.value.cmd.do('forwardDelete')
  // 获取位置
  let startOffset = document.getSelection().focusOffset
  // 当 ${} 中存在已输入的匹配字符时，需要计算对应字符长度
  if (keywordObj.currentStr) {
    startOffset = document.getSelection().focusOffset - keywordObj.currentStr.length
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

</script>
<style lang="less">
body {
  background-color: #f3f4f7;
}

.box {
  width: 1020px;
  margin: 80px auto 0;
  position: relative;

  .w-e-text-container {
    min-height: 100px;
  }

  .tip-wrap {
    width: 600px;
    position: absolute;
    left: 20px;
    background-color: #fff;
    box-shadow: 0 1px 6px rgba(0, 0, 0, 0.2);
    border: 1px solid #ddd;
    border-radius: 4px;
    z-index: 10010;

    li {
      &:hover {
        background-color: #eee;
        cursor: pointer;
      }
    }
  }
}
</style>