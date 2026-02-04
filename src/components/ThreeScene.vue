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
  const AUTO_ROTATE_SPEED = 0;
  let rotateGroupRef = null;

  /** Part = component (parent group) that contains the mesh; all meshes in the same component share one id. */
  function getPartId(obj) {
	  if (!obj) return null;
	  const parent = obj.parent;
	  if (parent && parent !== scene) return parent.uuid;
	  return obj.uuid;
  }

  function getDisplayName(obj) {
	  if (!obj) return 'Unnamed';
	  const name = obj.name?.trim();
	  if (name) return name;
	  let current = obj.parent;
	  while (current && current !== scene) {
	    const n = current.name?.trim();
	    if (n) return n;
	    current = current.parent;
	  }
	  return 'Unnamed';
  }

  function getPartLabel(obj) {
	  const name = getDisplayName(obj);
	  return PART_LABELS[name] ?? name;
  }

  function getContainerSize() {
	const rect = container.value?.getBoundingClientRect();
	const w = rect?.width ?? 0;
	const h = rect?.height ?? 0;
	return {
	  width: w > 0 ? w : window.innerWidth,
	  height: h > 0 ? h : window.innerHeight,
	};
  }

  onMounted(() => {
	const { width, height } = getContainerSize();
  
	scene = new THREE.Scene();
	scene.background = new THREE.Color(0x111111); // match dark viewer section
  
	camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 1000);
	camera.position.set(3, 3, 6);
  
	renderer = new THREE.WebGLRenderer({ antialias: true });
	renderer.setSize(width, height);
	renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
	renderer.outputColorSpace = THREE.SRGBColorSpace;
	renderer.toneMapping = THREE.ACESFilmicToneMapping;
	renderer.toneMappingExposure = 1.35;
  
	container.value.appendChild(renderer.domElement);

	raycaster = new THREE.Raycaster();
	pointer = new THREE.Vector2();
  
	const hemiLight = new THREE.HemisphereLight(0xffffff, 0x888888, 1.35);
	hemiLight.position.set(0, 10, 0);
	scene.add(hemiLight);
  
	const dirLight = new THREE.DirectionalLight(0xffffff, 1.2);
	dirLight.position.set(5, 10, 7);
	scene.add(dirLight);
	const fillLight = new THREE.DirectionalLight(0xffffff, 0.4);
	fillLight.position.set(-4, 6, 4);
	scene.add(fillLight);
  
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

		// Print structure: layers, names, grouping
		function nodeToTree(obj, depth = 0) {
		  const type = obj.isMesh ? 'Mesh' : (obj.isGroup ? 'Group' : 'Object3D');
		  const name = obj.name?.trim() || '(no name)';
		  const entry = { name, type, uuid: obj.uuid };
		  if (obj.children && obj.children.length > 0) {
		    entry.children = obj.children.map((c) => nodeToTree(c, depth + 1));
		  }
		  return entry;
		}
		const structure = nodeToTree(model);
		console.log('Model structure (layers / grouping):', structure);
		// Indented string for quick read
		function treeToString(node, indent = '') {
		  const type = node.type || '?';
		  const name = node.name || '(no name)';
		  let s = indent + `[${type}] ${name}\n`;
		  if (node.children?.length) {
		    node.children.forEach((c) => { s += treeToString(c, indent + '  '); });
		  }
		  return s;
		}
		console.log('Model structure (text):\n' + treeToString(structure));

		const box = new THREE.Box3().setFromObject(model);
		const size = box.getSize(new THREE.Vector3()).length();
		const center = box.getCenter(new THREE.Vector3());

		// Center the design: wrap in a group and offset model so rotation is around bounding center
		const rotateGroup = new THREE.Group();
		rotateGroupRef = rotateGroup;
		model.position.sub(center);
		rotateGroup.add(model);
		scene.add(rotateGroup);

		controls.target.set(0, 0, 0);
		controls.update();

		camera.position.set(size * 0.6, size * 0.4, size * 0.6);
		camera.lookAt(0, 0, 0);
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
	const { width, height } = getContainerSize();
	camera.aspect = width / height;
	camera.updateProjectionMatrix();
	renderer.setSize(width, height);
  }
  
  function animate() {
	animationId = requestAnimationFrame(animate);
	if (rotateGroupRef) rotateGroupRef.rotation.y += AUTO_ROTATE_SPEED;
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
	background: rgba(0, 0, 0, 0.75);
	border: 1px solid rgba(255, 255, 255, 0.2);
	border-radius: 8px;
	box-shadow: 0 4px 20px rgba(0, 0, 0, 0.4);
	color: #fff;
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
	background: rgba(0, 0, 0, 0.6);
	border: 1px solid rgba(255, 255, 255, 0.12);
	border-radius: 10px;
	box-shadow: 0 4px 24px rgba(0, 0, 0, 0.3);
	backdrop-filter: blur(12px);
	pointer-events: none;
	font-size: 0.875rem;
	color: #fff;
  }
  .part-row {
	display: flex;
	align-items: baseline;
	gap: 8px;
	padding: 4px 0;
  }
  .part-row + .part-row {
	border-top: 1px solid rgba(255, 255, 255, 0.1);
  }
  .part-label {
	color: rgba(255, 255, 255, 0.6);
	font-weight: 600;
	min-width: 64px;
  }
  .part-value {
	color: #fff;
	font-weight: 500;
  }
  .part-selected .part-value {
	color: #fb923c;
  }
  .part-placeholder {
	color: rgba(255, 255, 255, 0.6);
	opacity: 0.9;
	padding: 4px 0;
  }
  </style>