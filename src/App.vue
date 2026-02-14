<template>
  <div class="app">
    <h1 class="title">2048</h1>
    <Game2048 />
  </div>
</template>

<script setup>
import Game2048 from './components/Game2048.vue'
</script>

<style scoped>
.app {
  display: flex;
  flex-direction: column;
  align-items: center;
  min-height: 100dvh;
  padding: 20px 10px;
  box-sizing: border-box;
}
.title {
  font-size: 48px;
  font-weight: 800;
  color: #776e65;
  margin: 0 0 16px;
}
</style>
