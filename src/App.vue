<template>
  <div class="app">
    <header class="header">
      <div class="header-inner">
        <h1 class="title">3D Viewer</h1>
        <p class="subtitle">Drag to rotate · Scroll to zoom · Click a part to select</p>
      </div>
    </header>
    <main class="main">
      <div class="viewer-frame">
        <ThreeScene />
      </div>
    </main>
  </div>
</template>

<script setup>
import ThreeScene from './components/ThreeScene.vue';
</script>

<style>
.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  background: var(--bg-page);
}
.header {
  background: var(--header-bg);
  color: var(--header-text);
  padding: 1rem 1.5rem;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
}
.header-inner {
  max-width: 1600px;
  margin: 0 auto;
}
.title {
  margin: 0;
  font-size: 1.5rem;
  font-weight: 700;
  letter-spacing: -0.02em;
}
.subtitle {
  margin: 0.35rem 0 0;
  font-size: 0.875rem;
  opacity: 0.9;
  font-weight: 400;
}
.main {
  flex: 1;
  display: flex;
  align-items: stretch;
  justify-content: center;
  width: 100%;
  padding: 0.75rem;
  min-height: 0;
}
.viewer-frame {
  width: 100%;
  max-width: 1600px;
  height: 100%;
  min-height: 480px;
  margin: 0 auto;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.08), 0 2px 8px rgba(249, 115, 22, 0.06);
  background: var(--viewer-bg);
  border: 1px solid rgba(251, 146, 60, 0.2);
}
</style>
