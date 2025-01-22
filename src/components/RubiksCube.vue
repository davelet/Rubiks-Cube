<template>
  <div ref="container"></div>
</template>

<script lang="ts" setup>
import { onMounted, ref } from 'vue'
import CreepCube from '../index'

const container = ref<HTMLElement | null>(null)

onMounted(() => {
  if (container.value) {
    new CreepCube()
  }
})
</script>

<style scoped>
div {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}
</style>