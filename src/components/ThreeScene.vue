<template>
	<div class="scene-wrapper">
	  <div ref="container" class="canvas-container"></div>
	  <div
	    v-if="selectedPartLabel"
	    class="label-above"
	    :style="{ left: labelX + 'px', top: labelY + 'px' }"
	  >
	    {{ selectedPartLabel }}
	  </div>
	  <div class="part-panel">
	    <div v-if="hoveredPartLabel" class="part-row part-hover">
	      <span class="part-label">Hover</span>
	      <span class="part-value">{{ hoveredPartLabel }}</span>
	    </div>
	    <div v-if="selectedPartLabel" class="part-row part-selected">
	      <span class="part-label">Selected</span>
	      <span class="part-value">{{ selectedPartLabel }}</span>
	    </div>
	    <div v-if="!hoveredPartLabel && !selectedPartLabel" class="part-placeholder">
	      Move over the model to see parts · Click to select
	    </div>
	  </div>
	</div>
  </template>
  
  <script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

  import modelUrl from '../assets/models/3dmodel.glb?url';

  const container = ref(null);
  const hoveredPartLabel = ref('');
  const selectedPartLabel = ref('');
  const labelX = ref(0);
  const labelY = ref(0);

  /** Map SketchUp/GLB node names to display labels. Add your part names after loading (see console). */
  const PART_LABELS = {
	  // 'NodeNameFromSketchUp': 'Building A',
	  // 'AnotherGroup': 'Tower North',
  };

  let scene, camera, renderer, controls, animationId;
  let raycaster, pointer, hoveredObject = null;
  let selectedPartId = null;
	let onPointerMove, updateHover, applyPartStates, onPointerClick, updateLabelPosition;
  const interactiveMeshes = [];
  const originalMaterials = new Map();
  const originalScales = new Map();
  const HOVER_COLOR = new THREE.Color(0xfb923c);
  const SELECTED_COLOR = new THREE.Color(0xea580c);
  const SELECTED_SCALE = 1.5; // scale up when selected (by unique name)
  const labelOffsetY = -40; // pixels above the selected part
  const box3 = new THREE.Box3();
  const centerWorld = new THREE.Vector3();
  const centerNDC = new THREE.Vector3();

  function getPartId(obj) {
	  if (!obj) return null;
	  const name = obj.name?.trim();
	  if (name) return name;
	  if (obj.parent && obj.parent !== scene) return getPartId(obj.parent);
	  return null;
  }

  function getPartLabel(obj) {
	  const id = getPartId(obj);
	  if (!id) return 'Unnamed';
	  return PART_LABELS[id] ?? id;
  }

  onMounted(() => {
	const width = window.innerWidth;
	const height = window.innerHeight;
  
	scene = new THREE.Scene();
	scene.background = new THREE.Color(0xfef7ed); // match page cream
  
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

		// Collect meshes, part names, original materials and scales
		const partNames = new Set();
		model.traverse((child) => {
		  if (child.isMesh) {
		    interactiveMeshes.push(child);
		    const id = getPartId(child);
		    if (id) partNames.add(id);
		    originalScales.set(child, child.scale.clone());
		    const mat = child.material;
		    if (Array.isArray(mat)) {
		      originalMaterials.set(child, mat.map((m) => (m.color ? m.color.clone() : null)));
		    } else if (mat && mat.color) {
		      originalMaterials.set(child, mat.color.clone());
		    }
		  }
		});
		console.log('Part names in model (use these in PART_LABELS):', [...partNames].sort());
  
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

	  hoveredObject = hit;
	  hoveredPartLabel.value = hit ? getPartLabel(hit) : '';
	  renderer.domElement.style.cursor = hit ? 'pointer' : 'grab';
	};

	onPointerClick = () => {
	  if (!hoveredObject) return;
	  const id = getPartId(hoveredObject);
	  if (id === selectedPartId) {
	    selectedPartId = null;
	    selectedPartLabel.value = '';
	  } else {
	    selectedPartId = id;
	    selectedPartLabel.value = getPartLabel(hoveredObject);
	  }
	};

	function setMeshColor(mesh, color) {
	  const mat = mesh.material;
	  if (Array.isArray(mat)) mat.forEach((m) => { if (m.color) m.color.copy(color); });
	  else if (mat && mat.color) mat.color.copy(color);
	}

	function restoreMeshMaterial(mesh) {
	  const orig = originalMaterials.get(mesh);
	  if (!orig) return;
	  const mat = mesh.material;
	  if (Array.isArray(mat) && Array.isArray(orig)) {
	    mat.forEach((m, i) => { if (m.color && orig[i]) m.color.copy(orig[i]); });
	  } else if (mat && mat.color && orig && orig.clone) mat.color.copy(orig);
	}

	applyPartStates = () => {
	  if (interactiveMeshes.length === 0) return;
	  const hoveredId = hoveredObject ? getPartId(hoveredObject) : null;
	  for (const mesh of interactiveMeshes) {
	    const id = getPartId(mesh);
	    const baseScale = originalScales.get(mesh);
	    if (!baseScale) continue;
	    if (id === selectedPartId) {
	      setMeshColor(mesh, SELECTED_COLOR);
	      mesh.scale.copy(baseScale).multiplyScalar(SELECTED_SCALE);
	    } else if (id === hoveredId) {
	      setMeshColor(mesh, HOVER_COLOR);
	      mesh.scale.copy(baseScale);
	    } else {
	      restoreMeshMaterial(mesh);
	      mesh.scale.copy(baseScale);
	    }
	  }
	};

	updateLabelPosition = () => {
	  if (!selectedPartId || !container.value) return;
	  const meshes = interactiveMeshes.filter((m) => getPartId(m) === selectedPartId);
	  if (meshes.length === 0) return;
	  box3.makeEmpty();
	  for (const m of meshes) box3.expandByObject(m);
	  box3.getCenter(centerWorld);
	  centerNDC.copy(centerWorld).project(camera);
	  const rect = renderer.domElement.getBoundingClientRect();
	  labelX.value = ((centerNDC.x + 1) / 2) * rect.width;
	  labelY.value = ((1 - centerNDC.y) / 2) * rect.height + labelOffsetY;
	};

	renderer.domElement.addEventListener('pointermove', onPointerMove);
	renderer.domElement.addEventListener('click', onPointerClick);
  
	window.addEventListener('resize', onWindowResize);
	animate();
  });
  
  onBeforeUnmount(() => {
	selectedPartId = null;
	hoveredObject = null;
	applyPartStates?.();
	renderer?.domElement?.removeEventListener('pointermove', onPointerMove);
	renderer?.domElement?.removeEventListener('click', onPointerClick);
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
	applyPartStates();
	updateLabelPosition();
	controls.update();
	renderer.render(scene, camera);
  }
  </script>
  
  <style scoped>
  .scene-wrapper {
	position: relative;
	width: 100%;
	height: 100%;
	min-height: 380px;
  }
  .canvas-container {
	width: 100%;
	height: 100%;
	overflow: hidden;
	border-radius: 12px;
  }
  .label-above {
	position: absolute;
	transform: translate(-50%, -100%);
	padding: 6px 14px;
	background: var(--card-bg, rgba(255, 255, 255, 0.98));
	border: 1px solid var(--accent, #ea580c);
	border-radius: 8px;
	box-shadow: 0 4px 12px rgba(0, 0, 0, 0.12);
	color: var(--text-primary, #431407);
	font-size: 0.9rem;
	font-weight: 600;
	white-space: nowrap;
	pointer-events: none;
	z-index: 5;
  }
  .part-panel {
	position: absolute;
	bottom: 16px;
	left: 16px;
	right: 16px;
	max-width: 320px;
	padding: 12px 16px;
	background: var(--card-bg, rgba(255, 255, 255, 0.95));
	border: 1px solid var(--card-border, rgba(251, 146, 60, 0.25));
	border-radius: 10px;
	box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
	backdrop-filter: blur(8px);
	pointer-events: none;
	font-size: 0.875rem;
  }
  .part-row {
	display: flex;
	align-items: baseline;
	gap: 8px;
	padding: 4px 0;
  }
  .part-row + .part-row {
	border-top: 1px solid rgba(251, 146, 60, 0.15);
  }
  .part-label {
	color: var(--text-secondary, #7c2d12);
	font-weight: 600;
	min-width: 64px;
  }
  .part-value {
	color: var(--text-primary, #431407);
	font-weight: 500;
  }
  .part-selected .part-value {
	color: var(--accent, #ea580c);
  }
  .part-placeholder {
	color: var(--text-secondary, #7c2d12);
	opacity: 0.85;
	padding: 4px 0;
  }
  </style>