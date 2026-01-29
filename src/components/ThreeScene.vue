<template>
	<div ref="container" class="canvas-container"></div>
  </template>
  
  <script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

  // Bundled asset URL – works in prod (avoids SPA fallback returning HTML for /models/...)
  import modelUrl from '../assets/models/3dmodel.glb?url';

  const container = ref(null);
  
  let scene, camera, renderer, controls, animationId;
  
  onMounted(() => {
	const width = window.innerWidth;
	const height = window.innerHeight;
  
	scene = new THREE.Scene();
	scene.background = new THREE.Color(0x111111);
  
	camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 1000);
	camera.position.set(3, 3, 6);
  
	renderer = new THREE.WebGLRenderer({ antialias: true });
	renderer.setSize(width, height);
	renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  
	container.value.appendChild(renderer.domElement);
  
	const hemiLight = new THREE.HemisphereLight(0xffffff, 0x444444, 1.0);
	hemiLight.position.set(0, 10, 0);
	scene.add(hemiLight);
  
	const dirLight = new THREE.DirectionalLight(0xffffff, 1);
	dirLight.position.set(5, 10, 7);
	scene.add(dirLight);
  
	controls = new OrbitControls(camera, renderer.domElement);
	controls.enableDamping = true;

	// Resolve relative URL against current page so it works on subpaths (e.g. /my-3d-site/)
	const base = window.location.href.replace(/[^/]*$/, '');
	const fullModelUrl = new URL(modelUrl, base).href;

	const loader = new GLTFLoader();
	loader.load(
	  fullModelUrl,
	  (gltf) => {
		const model = gltf.scene;
		scene.add(model);
  
		const box = new THREE.Box3().setFromObject(model);
		const size = box.getSize(new THREE.Vector3()).length();
		const center = box.getCenter(new THREE.Vector3());
  
		controls.target.copy(center);
		controls.update();
  
		camera.position.copy(center);
		camera.position.x += size * 0.6;
		camera.position.y += size * 0.4;
		camera.position.z += size * 0.6;
		camera.lookAt(center);
	  },
	  undefined,
	  (error) => {
	    console.error('Error loading GLB:', error);
	    console.error('Requested URL:', fullModelUrl, '→ if this returns HTML, host may be serving index.html for assets');
	  }
	);
  
	window.addEventListener('resize', onWindowResize);
	animate();
  });
  
  onBeforeUnmount(() => {
	cancelAnimationFrame(animationId);
	window.removeEventListener('resize', onWindowResize);
	controls?.dispose();
	renderer?.dispose();
  });
  
  function onWindowResize() {
	const width = window.innerWidth;
	const height = window.innerHeight;
	camera.aspect = width / height;
	camera.updateProjectionMatrix();
	renderer.setSize(width, height);
  }
  
  function animate() {
	animationId = requestAnimationFrame(animate);
	controls.update();
	renderer.render(scene, camera);
  }
  </script>
  
  <style scoped>
  .canvas-container {
	width: 100vw;
	height: 100vh;
	overflow: hidden;
  }
  </style>