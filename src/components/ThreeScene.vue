<script setup lang="ts">
import { ref, onMounted, watch, onUnmounted } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

const props = defineProps<{
  config: {
    mode: string;
    color: string;
    wireframe: boolean;
    geometry: string;
    scale: number;
    rotationSpeed: number;
    intensity: number;
    collisionAvoidance: boolean;
    showBuildings: boolean;
    dayNightCycle: boolean;
    isPaused: boolean;
    carSpeed: number;
    carCount: number;
    paths: {
      id: string;
      name: string;
      type: string;
      vehicleType: string;
      points: { x: number; z: number }[];
      carCount: number;
      speed: number;
      color: string;
      carSpeeds: number[];
    }[];
    activePathId: string;
    stats: {
      avgSpeed: string;
      density: string;
      throughput: string;
    };
  }
}>();

const emit = defineEmits(['update']);

const container = ref<HTMLDivElement | null>(null);
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let mesh: THREE.Mesh;
let cars: THREE.Group[] = [];
let map: THREE.Group;
let pathLines: Map<string, THREE.Object3D> = new Map();
let pathCars: Map<string, THREE.Group[]> = new Map();
let tunnelLights: Map<string, THREE.Mesh[]> = new Map();
let buildings: THREE.Group;
let sun: THREE.DirectionalLight;
let controls: OrbitControls;
let animationId: number;
let raycaster = new THREE.Raycaster();
let mouse = new THREE.Vector2();
const carProgressMap = new Map<string, Float32Array>();

const init = () => {
  if (!container.value) return;

  // Scene
  scene = new THREE.Scene();
  scene.background = new THREE.Color('#09090b');

  // Camera
  camera = new THREE.PerspectiveCamera(75, container.value.clientWidth / container.value.clientHeight, 0.1, 1000);
  camera.position.set(15, 15, 15);

  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setSize(container.value.clientWidth, container.value.clientHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  container.value.appendChild(renderer.domElement);

  // Controls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.enableZoom = false; // Disable default zoom to use custom zoom-to-mouse logic

  // Lights
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
  scene.add(ambientLight);

  sun = new THREE.DirectionalLight(0xffffff, 1);
  sun.position.set(10, 20, 10);
  sun.castShadow = true;
  scene.add(sun);

  // Initial Setup
  updateScene();

  // Animation
  let time = 0;
  let lastStatsUpdate = 0;
  
  const animate = () => {
    animationId = requestAnimationFrame(animate);
    
    if (props.config.isPaused) {
      controls.update();
      renderer.render(scene, camera);
      return;
    }

    time += 0.01;

    // Day/Night Cycle
    if (props.config.dayNightCycle) {
      const cycleSpeed = 0.1;
      const sunAngle = time * cycleSpeed;
      sun.position.x = Math.cos(sunAngle) * 20;
      sun.position.y = Math.sin(sunAngle) * 20;
      sun.intensity = Math.max(0, Math.sin(sunAngle)) * 1.5;
      
      const skyColor = new THREE.Color().setHSL(0.6, 0.2, Math.max(0.05, Math.sin(sunAngle) * 0.1));
      scene.background = skyColor;
      scene.fog = new THREE.Fog(skyColor, 10, 60);
    }
    
    if (props.config.mode === 'Object' && mesh) {
      mesh.rotation.x += props.config.rotationSpeed;
      mesh.rotation.y += props.config.rotationSpeed;
    } else if (props.config.mode === 'Car') {
      cars.forEach((car, i) => {
        const offset = (i / props.config.carCount) * Math.PI * 2;
        const angle = (time * props.config.carSpeed * 10) + offset;
        const radius = 4;
        car.position.x = Math.cos(angle) * radius;
        car.position.z = Math.sin(angle) * radius;
        car.rotation.y = -angle + Math.PI / 2;
      });
    } else if (props.config.mode === 'Path') {
      let totalSpeed = 0;
      let activeCarCount = 0;

      props.config.paths.forEach(path => {
        if (path.points.length < 2) return;

        // Animate tunnel lights
        if (path.type === 'Tunnel') {
          const lights = tunnelLights.get(path.id);
          if (lights) {
            lights.forEach((light, idx) => {
              const offset = idx * 0.5;
              const intensity = (Math.sin(time * 5 - offset) + 1) * 0.5;
              (light.material as THREE.MeshBasicMaterial).opacity = 0.3 + intensity * 0.7;
              light.scale.setScalar(0.8 + intensity * 0.4);
            });
          }
          const pLine = pathLines.get(path.id) as THREE.Mesh;
          if (pLine && pLine.material) {
             (pLine.material as THREE.MeshStandardMaterial).emissiveIntensity = 0.3 + Math.sin(time * 2) * 0.2;
          }
        }

        const curve = new THREE.CatmullRomCurve3(
          path.points.map(p => {
            let y = 0.05;
            if (path.type === 'Elevated') y = 3;
            return new THREE.Vector3(p.x, y, p.z);
          }),
          true
        );

        const progress = carProgressMap.get(path.id);
        const pathCarsList = pathCars.get(path.id);
        
        if (progress && pathCarsList) {
          pathCarsList.forEach((car, i) => {
            let speed = (path.carSpeeds?.[i] || path.speed) * 0.1;

            if (props.config.collisionAvoidance) {
              const nextIndex = (i + 1) % path.carCount;
              if (path.carCount > 1) {
                let diff = progress[nextIndex] - progress[i];
                if (diff < 0) diff += 1;
                if (diff < 0.15) speed *= (diff / 0.15);
              }
            }

            progress[i] = (progress[i] + speed) % 1;
            const pos = curve.getPointAt(progress[i]);
            const tangent = curve.getTangentAt(progress[i]);
            car.position.copy(pos);
            car.lookAt(pos.clone().add(tangent));

            totalSpeed += speed;
            activeCarCount++;
          });
        }
      });

      // Update Stats every 1 second
      if (Date.now() - lastStatsUpdate > 1000) {
        const avgSpeed = activeCarCount > 0 ? (totalSpeed / activeCarCount) * 100 : 0;
        const density = activeCarCount / 10; // Normalized density
        emit('update', { 
          ...props.config, 
          stats: { 
            avgSpeed: avgSpeed.toFixed(1), 
            density: density.toFixed(2),
            throughput: (avgSpeed * density).toFixed(1)
          } 
        });
        lastStatsUpdate = Date.now();
      }
    }
    
    controls.update();
    renderer.render(scene, camera);
  };
  animate();

  // Events
  window.addEventListener('resize', onResize);
  container.value.addEventListener('mousedown', onMouseDown);
  container.value.addEventListener('mousemove', onMouseMove);
  container.value.addEventListener('wheel', onWheel, { passive: false });
};

const onMouseMove = (event: MouseEvent) => {
  if (!container.value) return;
  const rect = container.value.getBoundingClientRect();
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
};

const onWheel = (event: WheelEvent) => {
  if (!container.value || !controls) return;
  
  // Prevent default scrolling
  event.preventDefault();

  // Raycast to find the point under the mouse
  raycaster.setFromCamera(mouse, camera);
  const intersects = raycaster.intersectObjects(scene.children, true);
  
  // We prefer the ground or buildings as the zoom target
  const targetIntersect = intersects.find(i => i.object.name === 'ground' || i.object.parent === buildings);
  
  if (targetIntersect) {
    const zoomSpeed = 0.1;
    const delta = event.deltaY > 0 ? 1 + zoomSpeed : 1 - zoomSpeed;
    
    // Calculate vector from camera to intersection point
    const zoomPoint = targetIntersect.point;
    const direction = new THREE.Vector3().subVectors(camera.position, zoomPoint);
    
    // Apply zoom by scaling the distance
    direction.multiplyScalar(delta);
    
    // Update camera position
    camera.position.copy(zoomPoint).add(direction);
    
    // Update controls target to keep the zoom point stable
    // We nudge the target towards the zoom point to make it feel more natural
    controls.target.lerp(zoomPoint, 0.2);
  }
};

const onMouseDown = (event: MouseEvent) => {
  if (props.config.mode !== 'Path' || !container.value) return;

  const rect = container.value.getBoundingClientRect();
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

  raycaster.setFromCamera(mouse, camera);
  const intersects = raycaster.intersectObjects(scene.children, true);

  const groundIntersect = intersects.find(i => i.object.name === 'ground');
  if (groundIntersect) {
    const activePath = props.config.paths.find(p => p.id === props.config.activePathId);
    if (activePath) {
      const newPoints = [...activePath.points, { x: groundIntersect.point.x, z: groundIntersect.point.z }];
      const newPaths = props.config.paths.map(p => 
        p.id === props.config.activePathId ? { ...p, points: newPoints } : p
      );
      emit('update', { ...props.config, paths: newPaths });
    }
  }
};

const updateScene = () => {
  // Clear existing
  if (mesh) scene.remove(mesh);
  cars.forEach(car => scene.remove(car));
  cars = [];
  pathCars.forEach(list => list.forEach(c => scene.remove(c)));
  pathCars.clear();
  tunnelLights.clear();
  if (map) scene.remove(map);
  if (buildings) scene.remove(buildings);
  pathLines.forEach(line => scene.remove(line));
  pathLines.clear();

  if (props.config.mode === 'Object') {
    createMesh();
    camera.position.set(0, 0, 5);
  } else if (props.config.mode === 'Car') {
    createMap();
    for (let i = 0; i < props.config.carCount; i++) {
      createCar(props.config.color, 'Car');
    }
    camera.position.set(12, 12, 12);
  } else if (props.config.mode === 'Path') {
    createMap();
    if (props.config.showBuildings) createEnvironment();
    props.config.paths.forEach(path => {
      createPathCars(path);
      updatePathLine(path);
    });
    camera.position.set(18, 18, 18);
  }
  controls.target.set(0, 0, 0);
};

const createEnvironment = () => {
  buildings = new THREE.Group();
  const boxGeo = new THREE.BoxGeometry(1, 1, 1);
  
  const count = 10 + Math.floor(Math.random() * 6); // 10 to 15
  for (let i = 0; i < count; i++) {
    const h = 2 + Math.random() * 6;
    const w = 1 + Math.random() * 1.5;
    const mat = new THREE.MeshStandardMaterial({ 
      color: '#999999',
      roughness: 0.5,
      metalness: 0.5
    });
    const b = new THREE.Mesh(boxGeo, mat);
    
    // Random position avoiding central area and staying within map (30x30 ground)
    let x, z;
    do {
      x = (Math.random() - 0.5) * 28; // Range [-14, 14]
      z = (Math.random() - 0.5) * 28; // Range [-14, 14]
    } while (Math.abs(x) < 6 && Math.abs(z) < 6);
    
    b.position.set(x, h/2, z);
    b.scale.set(w, h, w);
    buildings.add(b);

    // Add windows structured in rows/cols
    const winGeo = new THREE.PlaneGeometry(0.15, 0.15);
    const winMat = new THREE.MeshBasicMaterial({ 
      color: '#fde047',
      transparent: true,
      opacity: 0.8
    });
    
    // Add windows on 4 sides
    const rows = Math.floor(h * 2);
    const cols = 2;
    
    for (let r = 0; r < rows; r++) {
      for (let c = 0; c < cols; c++) {
        if (Math.random() > 0.4) { // Randomly skip some windows
          const win = new THREE.Mesh(winGeo, winMat);
          const yPos = (r / rows) - 0.45; // Local Y from -0.45 to 0.45
          const xOffset = (c - (cols-1)/2) * 0.3;
          
          // Front side
          const winFront = win.clone();
          winFront.position.set(xOffset, yPos, 0.51);
          b.add(winFront);
          
          // Back side
          const winBack = win.clone();
          winBack.position.set(xOffset, yPos, -0.51);
          winBack.rotation.y = Math.PI;
          b.add(winBack);
        }
      }
    }
  }
  scene.add(buildings);
};

const createMesh = () => {
  if (mesh) {
    scene.remove(mesh);
    mesh.geometry.dispose();
    if (Array.isArray(mesh.material)) {
      mesh.material.forEach(m => m.dispose());
    } else {
      mesh.material.dispose();
    }
  }

  let geometry;
  switch (props.config.geometry) {
    case 'Sphere': geometry = new THREE.SphereGeometry(1.5, 32, 32); break;
    case 'Torus': geometry = new THREE.TorusGeometry(1, 0.4, 16, 100); break;
    case 'Knot': geometry = new THREE.TorusKnotGeometry(1, 0.3, 100, 16); break;
    default: geometry = new THREE.BoxGeometry(2, 2, 2);
  }

  const material = new THREE.MeshStandardMaterial({
    color: props.config.color,
    wireframe: props.config.wireframe,
    roughness: 0.3,
    metalness: 0.7,
  });

  mesh = new THREE.Mesh(geometry, material);
  mesh.scale.set(props.config.scale, props.config.scale, props.config.scale);
  scene.add(mesh);
};

const createMap = () => {
  map = new THREE.Group();
  
  const groundGeo = new THREE.PlaneGeometry(30, 30);
  const groundMat = new THREE.MeshStandardMaterial({ color: '#111111', roughness: 0.9 });
  const ground = new THREE.Mesh(groundGeo, groundMat);
  ground.rotation.x = -Math.PI / 2;
  ground.name = 'ground';
  map.add(ground);

  const grid = new THREE.GridHelper(30, 30, '#333333', '#222222');
  grid.position.y = 0.01;
  map.add(grid);

  if (props.config.mode === 'Car') {
    const roadGeo = new THREE.TorusGeometry(4, 0.4, 16, 100);
    const roadMat = new THREE.MeshStandardMaterial({ color: '#222222' });
    const road = new THREE.Mesh(roadGeo, roadMat);
    road.rotation.x = Math.PI / 2;
    road.position.y = 0.02;
    map.add(road);
  }

  scene.add(map);
};

const createPathCars = (path: any) => {
  const carList: THREE.Group[] = [];
  const progress = new Float32Array(path.carCount);
  
  for (let i = 0; i < path.carCount; i++) {
    const car = createCarInstance(path.color, path.vehicleType || 'Car');
    scene.add(car);
    carList.push(car);
    progress[i] = i / path.carCount;
  }
  
  pathCars.set(path.id, carList);
  carProgressMap.set(path.id, progress);
};

const createCarInstance = (color: string, type: string = 'Car') => {
  const carGroup = new THREE.Group();
  
  let bodyW = 0.8, bodyH = 0.4, bodyD = 0.4;
  let cabinW = 0.4, cabinH = 0.3, cabinD = 0.3;
  
  if (type === 'Bus') {
    bodyW = 1.6; bodyH = 0.6; bodyD = 0.5;
    cabinW = 0.2; cabinH = 0.4; cabinD = 0.4;
  } else if (type === 'Truck') {
    bodyW = 1.4; bodyH = 0.8; bodyD = 0.6;
    cabinW = 0.4; cabinH = 0.5; cabinD = 0.5;
  }

  const bodyGeo = new THREE.BoxGeometry(bodyW, bodyH, bodyD);
  const bodyMat = new THREE.MeshStandardMaterial({ color });
  const body = new THREE.Mesh(bodyGeo, bodyMat);
  body.position.y = bodyH / 2 + 0.1;
  carGroup.add(body);

  const cabinGeo = new THREE.BoxGeometry(cabinW, cabinH, cabinD);
  const cabinMat = new THREE.MeshStandardMaterial({ color: '#ffffff', transparent: true, opacity: 0.5 });
  const cabin = new THREE.Mesh(cabinGeo, cabinMat);
  
  if (type === 'Bus') {
    cabin.position.set(0.7, bodyH/2 + 0.2, 0);
  } else if (type === 'Truck') {
    cabin.position.set(0.5, bodyH/2 + 0.3, 0);
  } else {
    cabin.position.set(-0.1, 0.6, 0);
  }
  carGroup.add(cabin);

  const wheelGeo = new THREE.CylinderGeometry(0.12, 0.12, 0.08, 16);
  const wheelMat = new THREE.MeshStandardMaterial({ color: '#000000' });
  
  const wheelPositions = type === 'Bus' || type === 'Truck' 
    ? [[-0.6, 0.12, 0.25], [-0.6, 0.12, -0.25], [0, 0.12, 0.25], [0, 0.12, -0.25], [0.6, 0.12, 0.25], [0.6, 0.12, -0.25]]
    : [[-0.25, 0.12, 0.2], [-0.25, 0.12, -0.2], [0.25, 0.12, 0.2], [0.25, 0.12, -0.2]];

  wheelPositions.forEach(pos => {
    const wheel = new THREE.Mesh(wheelGeo, wheelMat);
    wheel.rotation.x = Math.PI / 2;
    wheel.position.set(pos[0], pos[1], pos[2]);
    carGroup.add(wheel);
  });
  
  return carGroup;
};

const createCar = (color: string, type: string) => {
  const car = createCarInstance(color, type);
  cars.push(car);
  scene.add(car);
};

const updatePathLine = (path: any) => {
  if (path.points.length < 2) return;

  const y = path.type === 'Elevated' ? 3 : 0.05;
  const curve = new THREE.CatmullRomCurve3(
    path.points.map((p: any) => new THREE.Vector3(p.x, y, p.z)),
    true
  );

  let pLine: THREE.Object3D;

  if (path.type === 'Tunnel') {
    const tubeGeo = new THREE.TubeGeometry(curve, 128, 1.2, 12, true);
    // Improved Tunnel Visuals - Cyberpunk/Sci-fi style
    const tubeMat = new THREE.MeshStandardMaterial({ 
      color: '#020617', 
      transparent: true, 
      opacity: 0.7,
      side: THREE.DoubleSide,
      wireframe: true,
      emissive: '#3b82f6',
      emissiveIntensity: 0.5,
      metalness: 0.9,
      roughness: 0.1
    });
    pLine = new THREE.Mesh(tubeGeo, tubeMat);
    
    // Add internal lights effect
    const lights: THREE.Mesh[] = [];
    const points = curve.getPoints(40);
    points.forEach((pt, idx) => {
      // Create a light ring or point
      const lightGeo = new THREE.SphereGeometry(0.12, 12, 12);
      const lightMat = new THREE.MeshBasicMaterial({ 
        color: idx === 0 || idx === points.length - 1 ? '#ef4444' : '#60a5fa',
        transparent: true,
        opacity: 0.8
      });
      const light = new THREE.Mesh(lightGeo, lightMat);
      
      // Position light on the "ceiling" of the tunnel
      const tangent = curve.getTangentAt(idx / points.length);
      const up = new THREE.Vector3(0, 1, 0);
      const normal = new THREE.Vector3().crossVectors(tangent, up).normalize();
      const binormal = new THREE.Vector3().crossVectors(tangent, normal).normalize();
      
      light.position.copy(pt).add(binormal.clone().multiplyScalar(0.9));
      
      pLine.add(light);
      lights.push(light);

      // Add actual point lights for every 5th marker to save performance
      if (idx % 8 === 0) {
        const pLight = new THREE.PointLight(idx === 0 || idx === points.length - 1 ? '#ef4444' : '#3b82f6', 2, 5);
        pLight.position.copy(light.position);
        pLine.add(pLight);
      }
    });
    tunnelLights.set(path.id, lights);
  } else {
    const points = curve.getPoints(200);
    const geometry = new THREE.BufferGeometry().setFromPoints(points);
    const material = new THREE.LineBasicMaterial({ 
      color: path.type === 'Elevated' ? '#71717a' : '#10b981', 
      linewidth: 4 
    });
    pLine = new THREE.Line(geometry, material);

    if (path.type === 'Elevated') {
      path.points.forEach((p: any) => {
        const pillarGroup = new THREE.Group();
        
        // Main Pillar
        const pillarGeo = new THREE.CylinderGeometry(0.18, 0.22, 3, 16);
        const pillarMat = new THREE.MeshStandardMaterial({ 
          color: '#3f3f46',
          metalness: 0.6,
          roughness: 0.4
        });
        const pillar = new THREE.Mesh(pillarGeo, pillarMat);
        pillar.position.y = 1.5;
        pillarGroup.add(pillar);

        // Pillar Base
        const baseGeo = new THREE.CylinderGeometry(0.35, 0.45, 0.4, 16);
        const baseMat = new THREE.MeshStandardMaterial({ color: '#18181b' });
        const base = new THREE.Mesh(baseGeo, baseMat);
        base.position.y = 0.2;
        pillarGroup.add(base);

        // Glowing Ring
        const ringGeo = new THREE.TorusGeometry(0.22, 0.03, 8, 24);
        const ringMat = new THREE.MeshBasicMaterial({ color: '#6366f1' });
        const ring = new THREE.Mesh(ringGeo, ringMat);
        ring.rotation.x = Math.PI / 2;
        ring.position.y = 2.2;
        pillarGroup.add(ring);

        // Small light on the ring
        const pLight = new THREE.PointLight('#6366f1', 0.5, 2);
        pLight.position.y = 2.2;
        pillarGroup.add(pLight);

        pillarGroup.position.set(p.x, 0, p.z);
        pLine.add(pillarGroup);
      });
    }
  }
  
  pathLines.set(path.id, pLine);
  scene.add(pLine);
};

const onResize = () => {
  if (!container.value) return;
  camera.aspect = container.value.clientWidth / container.value.clientHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(container.value.clientWidth, container.value.clientHeight);
};

watch(() => props.config.mode, updateScene);
watch(() => props.config.carCount, updateScene);
watch(() => props.config.showBuildings, updateScene);
watch(() => props.config.dayNightCycle, (val) => {
  if (!val) {
    // Reset to default day light
    sun.position.set(10, 20, 10);
    sun.intensity = 1.5;
    scene.background = new THREE.Color('#09090b');
    scene.fog = null;
  }
});
watch(() => props.config.paths, updateScene, { deep: true });
watch(() => props.config.geometry, createMesh);
watch(() => props.config.color, (val) => {
  if (props.config.mode === 'Object' && mesh) {
    (mesh.material as THREE.MeshStandardMaterial).color.set(val);
  } else {
    cars.forEach(car => {
      const body = car.children[0] as THREE.Mesh;
      (body.material as THREE.MeshStandardMaterial).color.set(val);
    });
  }
});
watch(() => props.config.wireframe, (val) => {
  if (mesh) (mesh.material as THREE.MeshStandardMaterial).wireframe = val;
});
watch(() => props.config.scale, (val) => {
  if (mesh) mesh.scale.set(val, val, val);
});

onMounted(init);
onUnmounted(() => {
  cancelAnimationFrame(animationId);
  window.removeEventListener('resize', onResize);
  if (container.value) {
    container.value.removeEventListener('mousedown', onMouseDown);
    container.value.removeEventListener('mousemove', onMouseMove);
    container.value.removeEventListener('wheel', onWheel);
  }
  if (renderer) {
    renderer.dispose();
  }
});
</script>

<template>
  <div ref="container" class="w-full h-full"></div>
</template>
