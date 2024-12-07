<template>
  <div class="canvas-container">
    <v-stage :config="stageConfig" @wheel="handleWheel" @mouseup="handleMouseUp" @mouseleave="handleMouseUp">
      <!-- 背景层 -->
      <v-layer>
        <v-rect :config="backgroundConfig" />
      </v-layer>

      <!-- 头部标题层 -->
      <v-layer>
        <v-text :config="{
          x: 10,
          y: 10,
          text: 'Offset',
          fontSize: 14,
          fontFamily: 'Consolas, monospace',
          fill: '#666'
        }" />
        
        <v-text :config="{
          x: 100,
          y: 10,
          text: '00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F',
          fontSize: 14,
          fontFamily: 'Consolas, monospace',
          fill: '#666'
        }" />

        <v-text :config="{
          x: 520,
          y: 10,
          text: 'ASCII',
          fontSize: 14,
          fontFamily: 'Consolas, monospace',
          fill: '#666'
        }" />
      </v-layer>

      <!-- 选中高亮背景层 -->
      <v-layer>
        <v-rect
          v-for="index in selectedIndexes"
          :key="'hex_bg_' + index"
          :config="{
            x: 100 + (index % 16) * 25 - 2,
            y: 40 + Math.floor((index - startOffset) / 16) * 20 - 2,
            width: 24,
            height: 18,
            fill: '#3584e4',
            opacity: 0.3
          }"
        />
        <v-rect
          v-for="index in selectedIndexes"
          :key="'ascii_bg_' + index"
          :config="{
            x: 520 + (index % 16) * 10 - 2,
            y: 40 + Math.floor((index - startOffset) / 16) * 20 - 2,
            width: 12,
            height: 18,
            fill: '#3584e4',
            opacity: 0.3
          }"
        />
      </v-layer>

      <!-- 数据显示层 -->
      <v-layer ref="dataLayer">
        <!-- 地址列 -->
        <v-text
          v-for="row in visibleRows"
          :key="'addr_' + row"
          :config="{
            x: 10,
            y: 40 + row * 20,
            text: getAddressText(row),
            fontSize: 14,
            fontFamily: 'Consolas, monospace',
            fill: '#666'
          }"
        />

        <!-- 十六进制区域 -->
        <v-text
          v-for="(byte, index) in visibleHexBytes"
          :key="'hex_' + index"
          :config="{
            x: 100 + (index % 16) * 25,
            y: 40 + Math.floor(index / 16) * 20,
            text: byte,
            fontSize: 14,
            fontFamily: 'Consolas, monospace',
            fill: '#000',
            width: 20,
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
            x: 520 + (index % 16) * 10,
            y: 40 + Math.floor(index / 16) * 20,
            text: char,
            fontSize: 14,
            fontFamily: 'Consolas, monospace',
            fill: '#000'
          }"
          @mousedown="handleMouseDown($event, startOffset + index)"
          @mousemove="handleMouseMove($event, startOffset + index)"
        />
      </v-layer>

      <!-- 分隔线 -->
      <v-layer>
        <v-line :config="{
          points: [500, 0, 500, stageConfig.height],
          stroke: '#ccc',
          strokeWidth: 1
        }" />
      </v-layer>
    </v-stage>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, computed } from 'vue'

export default defineComponent({
  name: 'HexViewer',
  props: {
    data: {
      type: Uint8Array,
      required: true
    }
  },
  setup(props) {
    // 配置
    const stageConfig = ref({
      width: 700,
      height: 600
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
    const bytesPerRow = 16
    const visibleRows = ref(Math.floor((stageConfig.value.height - 40) / 20))

    // 计算可见的字节数据
    const visibleHexBytes = computed(() => {
      const bytes: string[] = []
      const start = startOffset.value
      const end = Math.min(start + visibleRows.value * bytesPerRow, props.data.length)
      
      for (let i = start; i < end; i++) {
        bytes.push(props.data[i].toString(16).padStart(2, '0').toUpperCase())
      }
      return bytes
    })

    // 计算可见的ASCII字符
    const visibleAsciiChars = computed(() => {
      const chars: string[] = []
      const start = startOffset.value
      const end = Math.min(start + visibleRows.value * bytesPerRow, props.data.length)
      
      for (let i = start; i < end; i++) {
        const byte = props.data[i]
        chars.push(byte >= 32 && byte <= 126 ? String.fromCharCode(byte) : '.')
      }
      return chars
    })

    // 获取地址文本
    const getAddressText = (row: number) => {
      const address = startOffset.value + row * bytesPerRow
      return address.toString(16).padStart(8, '0').toUpperCase()
    }

    // 更新选择范围
    const updateSelection = (start: number, end: number) => {
      const newSelection: number[] = []
      const [min, max] = start <= end ? [start, end] : [end, start]
      
      for (let i = min; i <= max; i++) {
        if (i >= 0 && i < props.data.length) {
          newSelection.push(i)
        }
      }
      
      selectedIndexes.value = newSelection
    }

    // 事件处理
    const handleWheel = (e: any) => {
      e.evt.preventDefault()
      const delta = e.evt.deltaY > 0 ? bytesPerRow : -bytesPerRow
      const newOffset = Math.max(0, Math.min(startOffset.value + delta, props.data.length - visibleRows.value * bytesPerRow))
      startOffset.value = newOffset
    }

    const handleMouseDown = (e: any, index: number) => {
      isSelecting.value = true
      selectionStart.value = index
      
      if (!e.evt.shiftKey) {
        // 普通点击，开始新的选择
        selectedIndexes.value = [index]
      } else {
        // Shift点击，扩展选择
        const lastSelected = selectedIndexes.value[selectedIndexes.value.length - 1] || index
        updateSelection(lastSelected, index)
      }
    }

    const handleMouseMove = (e: any, index: number) => {
      if (isSelecting.value && selectionStart.value !== -1) {
        updateSelection(selectionStart.value, index)
      }
    }

    const handleMouseUp = () => {
      isSelecting.value = false
      selectionStart.value = -1
    }

    return {
      stageConfig,
      backgroundConfig,
      startOffset,
      visibleRows,
      visibleHexBytes,
      visibleAsciiChars,
      selectedIndexes,
      getAddressText,
      handleWheel,
      handleMouseDown,
      handleMouseMove,
      handleMouseUp
    }
  }
})
</script>

<style scoped>
.canvas-container {
  border: 1px solid #ccc;
  border-radius: 4px;
  display: inline-block;
  margin: 20px;
  background: #fff;
  user-select: none;
}
</style>
