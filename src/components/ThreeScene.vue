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

  // Hover: raycasting + restore original colors
  let scene, camera, renderer, controls, animationId;
  let raycaster, pointer, hoveredObject = null;
  let onPointerMove, updateHover, setHoverMaterial, restoreMaterial;
  const interactiveMeshes = [];
  const originalMaterials = new Map();
  const HOVER_COLOR = new THREE.Color(0x88ccff); // change to any hex, e.g. 0xff6600

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

	raycaster = new THREE.Raycaster();
	pointer = new THREE.Vector2();
  
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

		// Collect meshes for hover and store original materials
		model.traverse((child) => {
		  if (child.isMesh) {
		    interactiveMeshes.push(child);
		    const mat = child.material;
		    if (Array.isArray(mat)) {
		      originalMaterials.set(child, mat.map((m) => (m.color ? m.color.clone() : null)));
		    } else if (mat && mat.color) {
		      originalMaterials.set(child, mat.color.clone());
		    }
		  }
		});
  
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

	onPointerMove = (event) => {
	  const rect = renderer.domElement.getBoundingClientRect();
	  pointer.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
	  pointer.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
	};

	updateHover = () => {
	  if (interactiveMeshes.length === 0) return;
	  raycaster.setFromCamera(pointer, camera);
	  const intersects = raycaster.intersectObjects(interactiveMeshes, true);
	  const hit = intersects.length > 0 ? intersects[0].object : null;

	  if (hit !== hoveredObject) {
	    if (hoveredObject) restoreMaterial(hoveredObject);
	    hoveredObject = hit;
	    if (hoveredObject) setHoverMaterial(hoveredObject);
	    renderer.domElement.style.cursor = hit ? 'pointer' : 'grab';
	  }
	};

	setHoverMaterial = (mesh) => {
	  const mat = mesh.material;
	  if (Array.isArray(mat)) {
	    mat.forEach((m) => { if (m.color) m.color.copy(HOVER_COLOR); });
	  } else if (mat && mat.color) {
	    mat.color.copy(HOVER_COLOR);
	  }
	};

	restoreMaterial = (mesh) => {
	  const orig = originalMaterials.get(mesh);
	  if (!orig) return;
	  const mat = mesh.material;
	  if (Array.isArray(mat) && Array.isArray(orig)) {
	    mat.forEach((m, i) => { if (m.color && orig[i]) m.color.copy(orig[i]); });
	  } else if (mat && mat.color && orig && orig.clone) {
	    mat.color.copy(orig);
	  }
	};

	renderer.domElement.addEventListener('pointermove', onPointerMove);
  
	window.addEventListener('resize', onWindowResize);
	animate();
  });
  
  onBeforeUnmount(() => {
	if (hoveredObject) restoreMaterial(hoveredObject);
	renderer?.domElement?.removeEventListener('pointermove', onPointerMove);
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
	updateHover();
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