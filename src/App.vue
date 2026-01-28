<template>
  <div id="app">
    <header class="overlay-ui">
      <h1>My 3D Experience</h1>
    </header>
    <ThreeScene />
  </div>
</template>

<script setup>
import ThreeScene from './components/ThreeScene.vue';
</script>

<style>
html, body, #app {
  margin: 0;
  padding: 0;
  height: 100%;
}
#app {
  font-family: system-ui, sans-serif;
}
.overlay-ui {
  position: absolute;
  z-index: 10;
  color: white;
  padding: 16px;
}
</style>