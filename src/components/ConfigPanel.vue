<script setup lang="ts">
import { 
  Box, 
  Circle, 
  Settings2, 
  Palette, 
  Maximize2, 
  RotateCw, 
  Sun,
  Layers
} from 'lucide-vue-next';

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

const modes = [
  { label: '单体', value: 'Object' },
  { label: '环路', value: 'Car' },
  { label: '路径', value: 'Path' }
];
const geometries = [
  { label: '立方体', value: 'Box' },
  { label: '球体', value: 'Sphere' },
  { label: '圆环', value: 'Torus' },
  { label: '纽结', value: 'Knot' }
];
const pathTypes = [
  { label: '平面', value: 'Plane' },
  { label: '高架', value: 'Elevated' },
  { label: '隧道', value: 'Tunnel' }
];
const vehicleTypes = [
  { label: '轿车', value: 'Car' },
  { label: '巴士', value: 'Bus' },
  { label: '卡车', value: 'Truck' }
];

const activePath = computed(() => {
  return props.config.paths.find(p => p.id === props.config.activePathId) || props.config.paths[0];
});

const update = (key: string, value: any) => {
  emit('update', { ...props.config, [key]: value });
};

const updateActivePath = (key: string, value: any) => {
  const newPaths = props.config.paths.map(p => {
    if (p.id === props.config.activePathId) {
      const updated = { ...p, [key]: value };
      // Ensure carSpeeds array matches carCount
      if (key === 'carCount') {
        const count = value as number;
        const speeds = [...(updated.carSpeeds || [])];
        while (speeds.length < count) speeds.push(p.speed);
        updated.carSpeeds = speeds.slice(0, count);
      }
      return updated;
    }
    return p;
  });
  update('paths', newPaths);
};

const updateCarSpeed = (index: number, speed: number) => {
  const speeds = [...(activePath.value.carSpeeds || [])];
  speeds[index] = speed;
  updateActivePath('carSpeeds', speeds);
};

const addPath = () => {
  const id = `path-${Date.now()}`;
  const newPath = {
    id,
    name: `新路径 ${props.config.paths.length + 1}`,
    type: 'Plane',
    points: [],
    carCount: 2,
    speed: 0.05,
    color: '#' + Math.floor(Math.random()*16777215).toString(16),
    carSpeeds: [0.05, 0.05],
  };
  update('paths', [...props.config.paths, newPath]);
  update('activePathId', id);
};

const removePath = (id: string) => {
  if (props.config.paths.length <= 1) return;
  const newPaths = props.config.paths.filter(p => p.id !== id);
  update('paths', newPaths);
  if (props.config.activePathId === id) {
    update('activePathId', newPaths[0].id);
  }
};

const clearPath = () => {
  updateActivePath('points', []);
};

import { computed } from 'vue';
import { 
  Box, 
  Circle, 
  Settings2, 
  Palette, 
  Maximize2, 
  RotateCw, 
  Sun,
  Layers,
  Plus,
  Trash2,
  Car,
  Play,
  Pause
} from 'lucide-vue-next';
</script>

<template>
  <div class="p-6 space-y-8">
    <div class="flex items-center gap-2 text-zinc-400">
      <Settings2 :size="18" />
      <span class="text-xs font-bold uppercase tracking-widest">系统配置</span>
    </div>

    <!-- Mode Selection -->
    <div class="space-y-4">
      <div class="flex items-center justify-between">
        <div class="flex items-center gap-2 text-zinc-500">
          <Maximize2 :size="14" />
          <span class="text-[10px] font-bold uppercase tracking-wider">场景模式</span>
        </div>
        <button 
          @click="update('isPaused', !config.isPaused)"
          :class="[
            'flex items-center gap-1.5 px-2 py-1 rounded text-[10px] font-bold uppercase transition-all',
            config.isPaused 
              ? 'bg-amber-500/10 text-amber-500 border border-amber-500/20 hover:bg-amber-500/20' 
              : 'bg-emerald-500/10 text-emerald-500 border border-emerald-500/20 hover:bg-emerald-500/20'
          ]"
        >
          <component :is="config.isPaused ? Play : Pause" :size="12" />
          {{ config.isPaused ? '继续模拟' : '暂停模拟' }}
        </button>
      </div>
      <div class="grid grid-cols-3 gap-2">
        <button 
          v-for="m in modes" 
          :key="m.value"
          @click="update('mode', m.value)"
          :class="[
            'px-2 py-2 rounded-md text-[10px] font-bold transition-all border uppercase tracking-tighter',
            config.mode === m.value 
              ? 'bg-emerald-600 border-emerald-500 text-white shadow-lg shadow-emerald-500/20' 
              : 'bg-zinc-800/50 border-zinc-700 text-zinc-400 hover:border-zinc-600 hover:bg-zinc-800'
          ]"
        >
          {{ m.label }}
        </button>
      </div>
    </div>

    <!-- Environment Controls -->
    <div class="space-y-4 pt-4 border-t border-zinc-800">
      <div class="flex items-center gap-2 text-zinc-500">
        <Sun :size="14" />
        <span class="text-[10px] font-bold uppercase tracking-wider">环境设置</span>
      </div>
      <div class="grid grid-cols-2 gap-4">
        <label class="flex items-center justify-between group cursor-pointer">
          <span class="text-[10px] text-zinc-400 group-hover:text-zinc-200 transition-colors">建筑景观</span>
          <div class="relative inline-flex items-center cursor-pointer">
            <input type="checkbox" :checked="config.showBuildings" @change="(e) => update('showBuildings', (e.target as HTMLInputElement).checked)" class="sr-only peer">
            <div class="w-8 h-4 bg-zinc-800 peer-focus:outline-none rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-zinc-400 after:border-zinc-300 after:border after:rounded-full after:h-3 after:w-3 after:transition-all peer-checked:bg-emerald-600 peer-checked:after:bg-white"></div>
          </div>
        </label>
        <label class="flex items-center justify-between group cursor-pointer">
          <span class="text-[10px] text-zinc-400 group-hover:text-zinc-200 transition-colors">昼夜交替</span>
          <div class="relative inline-flex items-center cursor-pointer">
            <input type="checkbox" :checked="config.dayNightCycle" @change="(e) => update('dayNightCycle', (e.target as HTMLInputElement).checked)" class="sr-only peer">
            <div class="w-8 h-4 bg-zinc-800 peer-focus:outline-none rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-zinc-400 after:border-zinc-300 after:border after:rounded-full after:h-3 after:w-3 after:transition-all peer-checked:bg-emerald-600 peer-checked:after:bg-white"></div>
          </div>
        </label>
      </div>
    </div>

    <div v-if="config.mode === 'Object'" class="space-y-8">
      <!-- Geometry Selection -->
      <div class="space-y-4">
        <div class="flex items-center gap-2 text-zinc-500">
          <Layers :size="14" />
          <span class="text-[10px] font-bold uppercase tracking-wider">几何体形状</span>
        </div>
        <div class="grid grid-cols-2 gap-2">
          <button 
            v-for="geo in geometries" 
            :key="geo.value"
            @click="update('geometry', geo.value)"
            :class="[
              'px-3 py-2 rounded-md text-xs font-medium transition-all border',
              config.geometry === geo.value 
                ? 'bg-indigo-600 border-indigo-500 text-white shadow-lg shadow-indigo-500/20' 
                : 'bg-zinc-800/50 border-zinc-700 text-zinc-400 hover:border-zinc-600 hover:bg-zinc-800'
            ]"
          >
            {{ geo.label }}
          </button>
        </div>
      </div>

      <div class="space-y-6">
        <div class="flex items-center gap-2 text-zinc-500">
          <Palette :size="14" />
          <span class="text-[10px] font-bold uppercase tracking-wider">外观设置</span>
        </div>
        <div class="space-y-4">
          <div class="space-y-2">
            <div class="flex justify-between text-[10px] text-zinc-500 uppercase">
              <span>颜色</span>
              <span class="font-mono text-zinc-400">{{ config.color }}</span>
            </div>
            <input type="color" :value="config.color" @input="(e) => update('color', (e.target as HTMLInputElement).value)" class="w-full h-10 bg-zinc-800 border border-zinc-700 rounded-md cursor-pointer p-1" />
          </div>
          <label class="flex items-center justify-between group cursor-pointer">
            <span class="text-xs text-zinc-400 group-hover:text-zinc-200 transition-colors">线框模式</span>
            <div class="relative inline-flex items-center cursor-pointer">
              <input type="checkbox" :checked="config.wireframe" @change="(e) => update('wireframe', (e.target as HTMLInputElement).checked)" class="sr-only peer">
              <div class="w-9 h-5 bg-zinc-800 peer-focus:outline-none rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-zinc-400 after:border-zinc-300 after:border after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:bg-indigo-600 peer-checked:after:bg-white"></div>
            </div>
          </label>
        </div>
      </div>

      <!-- Transform -->
      <div class="space-y-6">
        <div class="flex items-center gap-2 text-zinc-500">
          <Maximize2 :size="14" />
          <span class="text-[10px] font-bold uppercase tracking-wider">变换参数</span>
        </div>

        <div class="space-y-6">
          <div class="space-y-2">
            <div class="flex justify-between text-[10px] text-zinc-500 uppercase">
              <span>缩放倍数</span>
              <span class="font-mono text-zinc-400">{{ config.scale.toFixed(2) }}</span>
            </div>
            <input 
              type="range" 
              min="0.1" 
              max="3" 
              step="0.1"
              :value="config.scale"
              @input="(e) => update('scale', parseFloat((e.target as HTMLInputElement).value))"
              class="w-full h-1.5 bg-zinc-800 rounded-lg appearance-none cursor-pointer accent-indigo-500"
            />
          </div>

          <div class="space-y-2">
            <div class="flex justify-between text-[10px] text-zinc-500 uppercase">
              <span>旋转速度</span>
              <span class="font-mono text-zinc-400">{{ config.rotationSpeed.toFixed(3) }}</span>
            </div>
            <input 
              type="range" 
              min="0" 
              max="0.1" 
              step="0.001"
              :value="config.rotationSpeed"
              @input="(e) => update('rotationSpeed', parseFloat((e.target as HTMLInputElement).value))"
              class="w-full h-1.5 bg-zinc-800 rounded-lg appearance-none cursor-pointer accent-indigo-500"
            />
          </div>
        </div>
      </div>
    </div>

    <div v-else-if="config.mode === 'Car'" class="space-y-8">
      <div class="space-y-6">
        <div class="flex items-center gap-2 text-zinc-500">
          <RotateCw :size="14" />
          <span class="text-[10px] font-bold uppercase tracking-wider">模拟参数</span>
        </div>
        <div class="space-y-4">
          <div class="space-y-2">
            <div class="flex justify-between text-[10px] text-zinc-500 uppercase">
              <span>行驶速度</span>
              <span class="font-mono text-zinc-400">{{ config.carSpeed.toFixed(2) }}</span>
            </div>
            <input type="range" min="0.01" max="0.5" step="0.01" :value="config.carSpeed" @input="(e) => update('carSpeed', parseFloat((e.target as HTMLInputElement).value))" class="w-full h-1.5 bg-zinc-800 rounded-lg appearance-none cursor-pointer accent-emerald-500" />
          </div>
          <div class="space-y-2">
            <div class="flex justify-between text-[10px] text-zinc-500 uppercase">
              <span>车辆数量</span>
              <span class="font-mono text-zinc-400">{{ config.carCount }}</span>
            </div>
            <input type="range" min="1" max="10" step="1" :value="config.carCount" @input="(e) => update('carCount', parseInt((e.target as HTMLInputElement).value))" class="w-full h-1.5 bg-zinc-800 rounded-lg appearance-none cursor-pointer accent-emerald-500" />
          </div>
          <div class="space-y-2">
            <div class="flex justify-between text-[10px] text-zinc-500 uppercase">
              <span>车辆颜色</span>
              <span class="font-mono text-zinc-400">{{ config.color }}</span>
            </div>
            <input type="color" :value="config.color" @input="(e) => update('color', (e.target as HTMLInputElement).value)" class="w-full h-10 bg-zinc-800 border border-zinc-700 rounded-md cursor-pointer p-1" />
          </div>
        </div>
      </div>
    </div>

    <div v-else-if="config.mode === 'Path'" class="space-y-8">
      <!-- Path Manager -->
      <div class="space-y-4">
        <div class="flex items-center justify-between">
          <div class="flex items-center gap-2 text-zinc-500">
            <Layers :size="14" />
            <span class="text-[10px] font-bold uppercase tracking-wider">路径管理</span>
          </div>
          <button @click="addPath" class="p-1 hover:bg-zinc-800 rounded text-emerald-400 transition-colors">
            <Plus :size="16" />
          </button>
        </div>
        <div class="space-y-2">
          <div 
            v-for="path in config.paths" 
            :key="path.id"
            @click="update('activePathId', path.id)"
            :class="[
              'flex items-center justify-between p-3 rounded-lg border cursor-pointer transition-all',
              config.activePathId === path.id 
                ? 'bg-zinc-800 border-emerald-500/50' 
                : 'bg-zinc-900/50 border-zinc-800 hover:border-zinc-700'
            ]"
          >
            <div class="flex items-center gap-3">
              <div class="w-2 h-2 rounded-full" :style="{ backgroundColor: path.color }"></div>
              <span class="text-xs font-medium" :class="config.activePathId === path.id ? 'text-white' : 'text-zinc-400'">{{ path.name }}</span>
            </div>
            <button 
              v-if="config.paths.length > 1"
              @click.stop="removePath(path.id)" 
              class="p-1 text-zinc-600 hover:text-red-400 transition-colors"
            >
              <Trash2 :size="14" />
            </button>
          </div>
        </div>
      </div>

      <div class="space-y-6">
        <div class="flex items-center gap-2 text-zinc-500">
          <Sun :size="14" />
          <span class="text-[10px] font-bold uppercase tracking-wider">当前路径设置: {{ activePath.name }}</span>
        </div>
        
        <div class="p-4 bg-emerald-500/10 border border-emerald-500/20 rounded-xl space-y-2">
          <p class="text-[10px] text-zinc-400 leading-relaxed">
            在 3D 网格上点击以放置航点。
          </p>
        </div>

        <div class="space-y-4">
          <div class="space-y-2">
            <div class="flex justify-between text-[10px] text-zinc-500 uppercase">
              <span>道路类型</span>
              <span class="font-mono text-zinc-400">{{ pathTypes.find(t => t.value === activePath.type)?.label }}</span>
            </div>
            <div class="grid grid-cols-3 gap-2">
              <button 
                v-for="t in pathTypes" 
                :key="t.value"
                @click="updateActivePath('type', t.value)"
                :class="[
                  'px-2 py-2 rounded-md text-[10px] font-bold transition-all border uppercase',
                  activePath.type === t.value 
                    ? 'bg-indigo-600 border-indigo-500 text-white' 
                    : 'bg-zinc-800/50 border-zinc-700 text-zinc-400 hover:border-zinc-600'
                ]"
              >
                {{ t.label }}
              </button>
            </div>
          </div>

          <div class="space-y-2">
            <div class="flex justify-between text-[10px] text-zinc-500 uppercase">
              <span>车辆类型</span>
              <span class="font-mono text-zinc-400">{{ vehicleTypes.find(v => v.value === activePath.vehicleType)?.label || '轿车' }}</span>
            </div>
            <div class="grid grid-cols-3 gap-2">
              <button 
                v-for="v in vehicleTypes" 
                :key="v.value"
                @click="updateActivePath('vehicleType', v.value)"
                :class="[
                  'px-2 py-2 rounded-md text-[10px] font-bold transition-all border uppercase',
                  activePath.vehicleType === v.value 
                    ? 'bg-emerald-600 border-emerald-500 text-white' 
                    : 'bg-zinc-800/50 border-zinc-700 text-zinc-400 hover:border-zinc-600'
                ]"
              >
                {{ v.label }}
              </button>
            </div>
          </div>

          <label class="flex items-center justify-between group cursor-pointer">
            <span class="text-xs text-zinc-400 group-hover:text-zinc-200 transition-colors">防碰撞系统</span>
            <div class="relative inline-flex items-center cursor-pointer">
              <input type="checkbox" :checked="config.collisionAvoidance" @change="(e) => update('collisionAvoidance', (e.target as HTMLInputElement).checked)" class="sr-only peer">
              <div class="w-9 h-5 bg-zinc-800 peer-focus:outline-none rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-zinc-400 after:border-zinc-300 after:border after:rounded-full after:h-4 after:w-4 after:transition-all peer-checked:bg-emerald-600 peer-checked:after:bg-white"></div>
            </div>
          </label>

          <div class="space-y-2">
            <div class="flex justify-between text-[10px] text-zinc-500 uppercase">
              <span>车辆数量</span>
              <span class="font-mono text-zinc-400">{{ activePath.carCount }}</span>
            </div>
            <input type="range" min="1" max="10" step="1" :value="activePath.carCount" @input="(e) => updateActivePath('carCount', parseInt((e.target as HTMLInputElement).value))" class="w-full h-1.5 bg-zinc-800 rounded-lg appearance-none cursor-pointer accent-emerald-500" />
          </div>

          <div class="space-y-2">
            <div class="flex justify-between text-[10px] text-zinc-500 uppercase">
              <span>基础速度</span>
              <span class="font-mono text-zinc-400">{{ activePath.speed.toFixed(2) }}</span>
            </div>
            <input type="range" min="0.01" max="0.5" step="0.01" :value="activePath.speed" @input="(e) => updateActivePath('speed', parseFloat((e.target as HTMLInputElement).value))" class="w-full h-1.5 bg-zinc-800 rounded-lg appearance-none cursor-pointer accent-emerald-500" />
          </div>

          <!-- Individual Car Speeds -->
          <div class="space-y-3 pt-2 border-t border-zinc-800">
            <div class="flex items-center gap-2 text-zinc-500">
              <Car :size="12" />
              <span class="text-[9px] font-bold uppercase tracking-wider">单车速度自定义</span>
            </div>
            <div class="space-y-3 max-h-48 overflow-y-auto pr-2">
              <div v-for="(s, i) in activePath.carSpeeds" :key="i" class="space-y-1">
                <div class="flex justify-between text-[9px] text-zinc-600 uppercase">
                  <span>车辆 #{{ i + 1 }}</span>
                  <span class="font-mono">{{ s.toFixed(2) }}</span>
                </div>
                <input 
                  type="range" 
                  min="0.01" 
                  max="0.5" 
                  step="0.01" 
                  :value="s" 
                  @input="(e) => updateCarSpeed(i, parseFloat((e.target as HTMLInputElement).value))" 
                  class="w-full h-1 bg-zinc-800 rounded-lg appearance-none cursor-pointer accent-indigo-400" 
                />
              </div>
            </div>
          </div>

          <button 
            @click="clearPath"
            class="w-full py-3 bg-zinc-800 border border-zinc-700 rounded-lg text-[10px] font-bold uppercase tracking-widest text-zinc-400 hover:bg-zinc-700 hover:text-white transition-all"
          >
            清除当前路径航点
          </button>
        </div>

        <div class="space-y-2">
          <div class="flex justify-between text-[10px] text-zinc-500 uppercase">
            <span>已放置航点</span>
            <span class="font-mono text-zinc-400">{{ activePath.points.length }}</span>
          </div>
          <div class="max-h-40 overflow-y-auto space-y-1 pr-2">
            <div v-for="(p, i) in activePath.points" :key="i" class="flex justify-between items-center p-2 bg-zinc-800/30 rounded border border-zinc-800 text-[9px] font-mono text-zinc-500">
              <span>航点 {{ i + 1 }}</span>
              <span>X: {{ p.x.toFixed(1) }} Z: {{ p.z.toFixed(1) }}</span>
            </div>
            <div v-if="activePath.points.length === 0" class="text-center py-4 text-[10px] text-zinc-600 italic">
              尚未添加任何点
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Info Card -->
    <div class="mt-auto pt-8 space-y-4">
      <!-- Traffic Stats Dashboard -->
      <div v-if="config.mode === 'Path' && config.stats" class="p-4 bg-zinc-900 border border-zinc-800 rounded-xl space-y-3">
        <div class="flex items-center gap-2 text-emerald-400">
          <RotateCw :size="14" />
          <span class="text-[10px] font-bold uppercase">实时仿真数据</span>
        </div>
        <div class="grid grid-cols-3 gap-2">
          <div class="p-2 bg-zinc-800/50 rounded-lg text-center">
            <p class="text-[8px] text-zinc-500 uppercase mb-1">平均车速</p>
            <p class="text-xs font-mono text-white">{{ config.stats.avgSpeed }}</p>
          </div>
          <div class="p-2 bg-zinc-800/50 rounded-lg text-center">
            <p class="text-[8px] text-zinc-500 uppercase mb-1">交通密度</p>
            <p class="text-xs font-mono text-white">{{ config.stats.density }}</p>
          </div>
          <div class="p-2 bg-zinc-800/50 rounded-lg text-center">
            <p class="text-[8px] text-zinc-500 uppercase mb-1">通行效率</p>
            <p class="text-xs font-mono text-white">{{ config.stats.throughput }}</p>
          </div>
        </div>
      </div>

      <div class="p-4 bg-indigo-600/10 border border-indigo-500/20 rounded-xl space-y-2">
        <div class="flex items-center gap-2 text-indigo-400">
          <Sun :size="14" />
          <span class="text-[10px] font-bold uppercase">操作提示</span>
        </div>
        <p class="text-[10px] text-zinc-400 leading-relaxed">
          使用鼠标左键旋转视角，右键平移，滚轮缩放。在路径模式下点击地面即可规划路线。
        </p>
      </div>
    </div>
  </div>
</template>

<style scoped>
input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 12px;
  height: 12px;
  background: #6366f1;
  border-radius: 50%;
  cursor: pointer;
}

input[type="range"]::-moz-range-thumb {
  width: 12px;
  height: 12px;
  background: #6366f1;
  border-radius: 50%;
  cursor: pointer;
}
</style>
