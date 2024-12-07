<template>
  <div class="canvas-container">
    <v-stage :config="stageConfig">
      <v-layer>
        <v-rect
          :config="{
            x: 50,
            y: 50,
            width: 100,
            height: 100,
            fill: 'red',
            draggable: true,
          }"
        />
      </v-layer>
    </v-stage>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref } from 'vue'

export default defineComponent({
  name: 'KonvaCanvas',
  setup() {
    const stageConfig = ref({
      width: 500,
      height: 500
    })

    return {
      stageConfig
    }
  }
})
</script>

<style scoped>
.canvas-container {
  border: 2px solid #333;
  border-radius: 4px;
  display: inline-block;
}
</style>
