<script setup lang="ts">
import { ref, reactive, onMounted } from 'vue';
import ThreeScene from './components/ThreeScene.vue';
import ConfigPanel from './components/ConfigPanel.vue';

const config = reactive({
  mode: 'Path', // 'Object', 'Car', 'Path'
  color: '#4f46e5',
  wireframe: false,
  geometry: 'Box',
  scale: 1,
  rotationSpeed: 0.01,
  intensity: 1,
  collisionAvoidance: true,
  showBuildings: true,
  dayNightCycle: true,
  isPaused: false,
  carSpeed: 0.05,
  carCount: 3,
  paths: [
    {
      id: 'path-1',
      name: '城市平面环路',
      type: 'Plane',
      vehicleType: 'Car', // 'Car', 'Bus', 'Truck'
      points: [
        { x: -8, z: -8 },
        { x: 8, z: -8 },
        { x: 8, z: 8 },
        { x: -8, z: 8 }
      ],
      carCount: 4,
      speed: 0.04,
      color: '#10b981',
      carSpeeds: [0.04, 0.05, 0.03, 0.04],
    },
    {
      id: 'path-2',
      name: '跨城高架桥',
      type: 'Elevated',
      vehicleType: 'Truck',
      points: [
        { x: -12, z: 0 },
        { x: 0, z: -12 },
        { x: 12, z: 0 },
        { x: 0, z: 12 }
      ],
      carCount: 3,
      speed: 0.06,
      color: '#71717a',
      carSpeeds: [0.06, 0.07, 0.05],
    },
    {
      id: 'path-3',
      name: '深海景观隧道',
      type: 'Tunnel',
      vehicleType: 'Bus',
      points: [
        { x: -5, z: -5 },
        { x: 5, z: -5 },
        { x: 5, z: 5 },
        { x: -5, z: 5 }
      ],
      carCount: 2,
      speed: 0.03,
      color: '#f59e0b',
      carSpeeds: [0.03, 0.04],
    }
  ],
  activePathId: 'path-1',
  stats: {
    avgSpeed: 0,
    density: 0,
    throughput: 0
  }
});

const updateConfig = (newConfig: any) => {
  Object.assign(config, newConfig);
};
</script>

<template>
  <div class="flex h-screen w-full bg-zinc-950 text-zinc-100 overflow-hidden font-sans">
    <!-- Sidebar / Config Panel -->
    <div class="w-80 border-r border-zinc-800 bg-zinc-900/50 backdrop-blur-xl z-10 overflow-y-auto">
      <ConfigPanel :config="config" @update="updateConfig" />
    </div>

    <!-- Main Viewport -->
    <div class="flex-1 relative">
      <div class="absolute top-6 left-6 z-10 pointer-events-none">
        <h1 class="text-2xl font-bold tracking-tighter text-white">3D 可视化配置系统</h1>
        <p class="text-xs text-zinc-500 font-mono uppercase tracking-widest mt-1">实时三维渲染引擎</p>
      </div>
      
      <ThreeScene :config="config" @update="updateConfig" />

      <div class="absolute bottom-6 right-6 z-10 flex gap-4">
        <div class="px-4 py-2 bg-zinc-900/80 border border-zinc-800 rounded-lg backdrop-blur-md text-[10px] font-mono text-zinc-400">
          帧率: <span class="text-emerald-400">60</span>
        </div>
        <div class="px-4 py-2 bg-zinc-900/80 border border-zinc-800 rounded-lg backdrop-blur-md text-[10px] font-mono text-zinc-400 uppercase tracking-wider">
          引擎: <span class="text-indigo-400">Three.js</span>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
body {
  margin: 0;
  padding: 0;
  overflow: hidden;
}
</style>
