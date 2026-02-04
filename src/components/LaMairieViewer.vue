<template>
  <div ref="container" class="viewer"></div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

const container = ref(null);
let scene, camera, renderer, controls, animationId;

function getContainerSize() {
  const rect = container.value?.getBoundingClientRect();
  const w = rect?.width ?? 0;
  const h = rect?.height ?? 0;
  return {
    width: w > 0 ? w : window.innerWidth,
    height: h > 0 ? h : window.innerHeight,
  };
}

function onWindowResize() {
  const { width, height } = getContainerSize();
  if (!camera || !renderer) return;
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  renderer.setSize(width, height);
}

onMounted(() => {
  const { width, height } = getContainerSize();
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0xfdeee0);
  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 1000);
  camera.position.set(2, 2, 4);
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(width, height);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 1.5;
  container.value.appendChild(renderer.domElement);

  const hemi = new THREE.HemisphereLight(0xffffff, 0xbbbbbb, 1.5);
  hemi.position.set(0, 10, 0);
  scene.add(hemi);
  const dir = new THREE.DirectionalLight(0xffffff, 1.0);
  dir.position.set(5, 10, 7);
  scene.add(dir);
  const fillLight = new THREE.DirectionalLight(0xffffff, 0.6);
  fillLight.position.set(-4, 6, 4);
  scene.add(fillLight);
  const backLight = new THREE.DirectionalLight(0xffffff, 0.35);
  backLight.position.set(0, 5, -5);
  scene.add(backLight);

  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;

  const base = window.location.href.replace(/[^/]*$/, '');
  const url = new URL('/models/la_mairie.glb', base).href;
  const loader = new GLTFLoader();
  loader.load(
    url,
    (gltf) => {
      const model = gltf.scene;
      scene.add(model);
      const box = new THREE.Box3().setFromObject(model);
      const center = box.getCenter(new THREE.Vector3());
      const size = box.getSize(new THREE.Vector3()).length();
      controls.target.copy(center);
      camera.position.copy(center).add(new THREE.Vector3(size * 0.6, size * 0.4, size * 0.6));
      camera.lookAt(center);
    },
    undefined,
    (e) => console.error('Error loading la_mairie.glb:', e)
  );

  window.addEventListener('resize', onWindowResize);

  function animate() {
    animationId = requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
  }
  animate();
});

onBeforeUnmount(() => {
  cancelAnimationFrame(animationId);
  window.removeEventListener('resize', onWindowResize);
  controls?.dispose();
  renderer?.dispose();
});
</script>

<style scoped>
.viewer {
  width: 100%;
  height: 100%;
  min-height: 320px;
  border-radius: 16px;
  overflow: hidden;
  background: #fdeee0;
}
</style>
