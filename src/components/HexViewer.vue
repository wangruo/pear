<template>
  <div class="canvas-container">
    <v-stage :config="stageConfig" @wheel="handleWheel" @mouseup="handleMouseUp" @mouseleave="handleMouseUp">
      <!-- 背景层 -->
      <v-layer>
        <v-rect :config="backgroundConfig" />
      </v-layer>

      <!-- 选中高亮背景层 -->
      <v-layer>
        <v-rect
          v-for="index in selectedIndexes"
          :key="'hex_bg_' + index"
          :config="{
            x: layout.hexColumnX + (index % layout.bytesPerRow) * layout.hexByteWidth - 0,
            y: layout.headerHeight + Math.floor((index - startOffset) / layout.bytesPerRow) * layout.rowHeight - layout.selectionPadding,
            width: layout.hexHighlightWidth,
            height: layout.highlightHeight,
            fill: layout.highlightColor,
            opacity: layout.highlightOpacity
          }"
        />
        <v-rect
          v-for="index in selectedIndexes"
          :key="'ascii_bg_' + index"
          :config="{
            x: layout.asciiColumnX + (index % layout.bytesPerRow) * layout.asciiCharWidth - layout.selectionPadding,
            y: layout.headerHeight + Math.floor((index - startOffset) / layout.bytesPerRow) * layout.rowHeight - layout.selectionPadding,
            width: layout.asciiHighlightWidth,
            height: layout.highlightHeight,
            fill: layout.highlightColor,
            opacity: layout.highlightOpacity
          }"
        />
      </v-layer>

      <!-- 数据显示层 -->
      <v-layer ref="dataLayer">
        <!-- 地址列 -->
        <v-text
          v-for="(_, row) in visibleRows"
          :key="'addr_' + row"
          :config="{
            x: layout.offsetColumnX,
            y: layout.headerHeight + row * layout.rowHeight,
            text: getAddressText(row),
            fontSize: layout.fontSize,
            fontFamily: layout.fontFamily,
            fill: layout.headerColor
          }"
        />

        <!-- 十六进制区域 -->
        <v-text
          v-for="(byte, index) in visibleHexBytes"
          :key="'hex_' + index"
          :config="{
            x: layout.hexColumnX + (index % layout.bytesPerRow) * layout.hexByteWidth,
            y: layout.headerHeight + Math.floor(index / layout.bytesPerRow) * layout.rowHeight,
            text: byte,
            fontSize: layout.fontSize,
            fontFamily: layout.fontFamily,
            fill: layout.textColor,
            width: layout.hexByteWidth,
            align: 'center'
          }"
          @mousedown="handleMouseDown($event, startOffset + index)"
          @mousemove="handleMouseMove($event, startOffset + index)"
        />

        <!-- ASCII 区域 -->
        <v-text
          v-for="(char, index) in visibleAsciiChars"
          :key="'ascii_' + index"
          :config="{
            x: layout.asciiColumnX + (index % layout.bytesPerRow) * layout.asciiCharWidth,
            y: layout.headerHeight + Math.floor(index / layout.bytesPerRow) * layout.rowHeight,
            text: char,
            fontSize: layout.fontSize,
            fontFamily: layout.fontFamily,
            fill: layout.textColor
          }"
          @mousedown="handleMouseDown($event, startOffset + index)"
          @mousemove="handleMouseMove($event, startOffset + index)"
        />
      </v-layer>

      <!-- 分隔线 -->
      <v-layer>
        <v-line :config="{
          points: [layout.asciiColumnX - layout.asciiCharWidth, 0, layout.asciiColumnX - layout.asciiCharWidth, stageConfig.height],
          stroke: layout.dividerColor,
          strokeWidth: 1
        }" />
      </v-layer>
    </v-stage>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'

// 布局常量
const layout = {
  // 视图尺寸
  viewWidth: 800,
  viewHeight: 600,
  
  // 基础布局
  headerHeight: 5, 
  rowHeight: 20,
  
  // 字体相关
  fontSize: 14,
  fontFamily: 'Consolas, monospace',
  
  // 列布局
  offsetColumnX: 10,
  hexColumnX: 100,
  asciiColumnX: 520,
  hexByteWidth: 25,
  asciiCharWidth: 13,
  
  // 选择高亮
  selectionPadding: 2,
  hexHighlightWidth: 25,
  asciiHighlightWidth: 13,
  highlightHeight: 18,
  highlightColor: '#3584e4',
  highlightOpacity: 0.3,
  
  // 数据显示
  bytesPerRow: 16,
  textColor: '#000',
  headerColor: '#666',
  dividerColor: '#ccc'
}

// 组件属性
interface Props {
  data: Uint8Array
}
const props = defineProps<Props>()

// 配置
const stageConfig = ref({
  width: layout.viewWidth,
  height: layout.viewHeight
})

const backgroundConfig = ref({
  x: 0,
  y: 0,
  width: stageConfig.value.width,
  height: stageConfig.value.height,
  fill: '#ffffff'
})

// 状态
const startOffset = ref(0)
const selectedIndexes = ref<number[]>([])
const isSelecting = ref(false)
const selectionStart = ref(-1)
const visibleRows = ref(Math.floor((stageConfig.value.height - layout.headerHeight) / layout.rowHeight))

// 计算可见的字节数据
const visibleHexBytes = computed(() => {
  const bytes: string[] = []
  const start = startOffset.value
  const end = Math.min(start + visibleRows.value * layout.bytesPerRow, props.data.length)
  
  for (let i = start; i < end; i++) {
    bytes.push(props.data[i].toString(16).padStart(2, '0').toUpperCase())
  }
  return bytes
})

// 计算可见的ASCII字符
const visibleAsciiChars = computed(() => {
  const chars: string[] = []
  const start = startOffset.value
  const end = Math.min(start + visibleRows.value * layout.bytesPerRow, props.data.length)
  
  for (let i = start; i < end; i++) {
    const byte = props.data[i]
    chars.push(byte >= 32 && byte <= 126 ? String.fromCharCode(byte) : '.')
  }
  return chars
})

// 获取地址文本
const getAddressText = (row: number) => {
  const address = startOffset.value + row * layout.bytesPerRow
  return address.toString(16).padStart(8, '0').toUpperCase()
}

// 更新选择范围
const updateSelection = (start: number, end: number) => {
  const newSelection: number[] = []
  const [min, max] = start <= end ? [start, end] : [end, start]
  
  for (let i = min; i <= max; i++) {
    newSelection.push(i)
  }
  selectedIndexes.value = newSelection
}

// 事件处理
const handleWheel = (e: any) => {
  e.evt.preventDefault()
  const delta = e.evt.deltaY > 0 ? layout.bytesPerRow : -layout.bytesPerRow
  const newOffset = Math.max(0, Math.min(startOffset.value + delta, props.data.length - visibleRows.value * layout.bytesPerRow))
  startOffset.value = newOffset
}

const handleMouseDown = (e: any, index: number) => {
  e.evt.preventDefault()
  isSelecting.value = true
  selectionStart.value = index
  updateSelection(index, index)
}

const handleMouseMove = (e: any, index: number) => {
  if (isSelecting.value && selectionStart.value !== -1) {
    updateSelection(selectionStart.value, index)
  }
}

const handleMouseUp = () => {
  isSelecting.value = false
}
</script>

<style scoped>
.canvas-container {
  background-color: #ffffff;
  border: 1px solid #ccc;
  border-radius: 4px;
  overflow: hidden;
}
</style>
