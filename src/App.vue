<template>
  <div class="app">
    <section class="hero" ref="heroRef">
      <div class="hero-bg"></div>
      <div class="hero-overlay"></div>
      <div class="hero-content">
        <h1 class="hero-title">Ready to explore Borj El Abhri Centre</h1>
        <p class="hero-subtitle">Step inside the 3D — discover every detail of your space</p>
        <button class="hero-cta" @click="scrollToViewer">
          Get Started
        </button>
      </div>
      <button class="hero-scroll-hint" @click="scrollToViewer" aria-label="Scroll to explorer">
        <span>Explore below</span>
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M12 5v14M19 12l-7 7-7-7"/>
        </svg>
      </button>
    </section>
    <section class="viewer-section" ref="viewerRef" :class="{ visible: viewerVisible }">
      <div class="viewer-wrapper">
        <p class="viewer-hint">Drag to rotate · Scroll to zoom · Click a part to select</p>
        <div class="viewer-frame">
          <ThreeScene />
        </div>
      </div>
    </section>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import ThreeScene from './components/ThreeScene.vue';

const heroRef = ref(null);
const viewerRef = ref(null);
const viewerVisible = ref(false);

function scrollToViewer() {
  viewerRef.value?.scrollIntoView({ behavior: 'smooth', block: 'start' });
  viewerVisible.value = true;
}

onMounted(() => {
  const el = viewerRef.value;
  if (!el) return;
  const observer = new IntersectionObserver(
    ([entry]) => {
      if (entry.isIntersecting) viewerVisible.value = true;
    },
    { threshold: 0.15 }
  );
  observer.observe(el);
});
</script>

<style>
.app {
  min-height: 100vh;
  background: #0a0a0a;
}
html {
  scroll-behavior: smooth;
}

/* Hero */
.hero {
  position: relative;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}
.hero-bg {
  position: absolute;
  inset: 0;
  background: url('/background.jpg') center / cover no-repeat;
  background-color: #1a1a1a;
}
.hero-overlay {
  position: absolute;
  inset: 0;
  background: linear-gradient(180deg, rgba(0,0,0,0.4) 0%, rgba(0,0,0,0.7) 100%);
  pointer-events: none;
}
.hero-content {
  position: relative;
  z-index: 1;
  text-align: center;
  padding: 2rem;
  max-width: 720px;
}
.hero-title {
  margin: 0;
  font-family: 'Plus Jakarta Sans', system-ui, sans-serif;
  font-size: clamp(2rem, 6vw, 3.5rem);
  font-weight: 700;
  letter-spacing: -0.03em;
  line-height: 1.15;
  color: #fff;
  text-shadow: 0 2px 24px rgba(0,0,0,0.4);
}
.hero-subtitle {
  margin: 1rem 0 2rem;
  font-size: clamp(1rem, 2.5vw, 1.25rem);
  font-weight: 400;
  color: rgba(255,255,255,0.85);
  letter-spacing: 0.02em;
}
.hero-cta {
  font-family: 'Plus Jakarta Sans', system-ui, sans-serif;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 1rem 2.25rem;
  font-size: 1rem;
  font-weight: 600;
  letter-spacing: 0.04em;
  color: #fff;
  background: rgba(255,255,255,0.12);
  border: 1px solid rgba(255,255,255,0.3);
  border-radius: 12px;
  cursor: pointer;
  backdrop-filter: blur(12px);
  transition: all 0.35s ease;
}
.hero-cta:hover {
  background: rgba(255,255,255,0.2);
  border-color: rgba(255,255,255,0.5);
  transform: translateY(-2px);
  box-shadow: 0 12px 40px rgba(0,0,0,0.3);
}
.hero-scroll-hint {
  position: absolute;
  bottom: 2rem;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem;
  background: none;
  border: none;
  color: rgba(255,255,255,0.6);
  font-size: 0.8rem;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  cursor: pointer;
  transition: color 0.2s;
}
.hero-scroll-hint:hover {
  color: rgba(255,255,255,0.95);
}
.hero-scroll-hint svg {
  animation: bounce 2s ease-in-out infinite;
}
@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(6px); }
}

/* Viewer section */
.viewer-section {
  min-height: 100vh;
  padding: 4rem 1.5rem 3rem;
  opacity: 0;
  transform: translateY(40px);
  transition: opacity 0.8s ease, transform 0.8s ease;
}
.viewer-section.visible {
  opacity: 1;
  transform: translateY(0);
}
.viewer-wrapper {
  max-width: 1400px;
  margin: 0 auto;
}
.viewer-hint {
  text-align: center;
  color: rgba(255,255,255,0.5);
  font-size: 0.875rem;
  margin-bottom: 1rem;
}
.viewer-frame {
  width: 100%;
  height: 70vh;
  min-height: 480px;
  border-radius: 16px;
  overflow: hidden;
  box-shadow: 0 24px 64px rgba(0,0,0,0.4), 0 0 0 1px rgba(255,255,255,0.06);
  background: #111;
  transform: scale(0.96);
  transition: transform 0.8s ease;
}
.viewer-section.visible .viewer-frame {
  transform: scale(1);
}
</style>
