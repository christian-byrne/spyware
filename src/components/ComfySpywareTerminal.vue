<script setup lang="ts">
import type { Patch } from '~/composables/spyware'
import { effectScope, onBeforeUnmount, onUnmounted, reactive, ref, computed } from 'vue'
import { spyware } from '~/composables/spyware'
import { useSpywareStore } from '~/composables/vueSpyware'

interface GraphNode {
  id: string
  title: string
  pos: [number, number]
  type: number
  size: [number, number]
  flags: { bypass?: boolean, mute?: boolean, collapsed?: boolean }
  order: number
  mode: number
  outputs: Array<{ name: string, type: string, links: string[] | null, slot_index?: number }>
  inputs?: Array<{ name: string, type: string, link: string | null }>
  properties: Record<string, any>
  widgets_values?: any[]
}

interface Widget {
  id: string
  nodeId: string
  name: string
  type: 'number' | 'text' | 'combo' | 'toggle'
  value: any
  options?: any
}

interface GraphLink {
  id: string
  origin_id: string
  origin_slot: number
  target_id: string
  target_slot: number
  type: string
}

interface GraphData {
  nodes: GraphNode[]
  links: GraphLink[]
  widgets: Widget[]
}

const demoScope = effectScope()

const initialGraph: GraphData = {
  nodes: [
    {
      id: '1',
      title: 'Load Image',
      pos: [100, 100],
      type: 1,
      size: [200, 150],
      flags: {},
      order: 0,
      mode: 0,
      outputs: [{ name: 'IMAGE', type: 'IMAGE', links: ['1'] }],
      properties: { filename: 'example.png' },
    },
    {
      id: '2',
      title: 'KSampler',
      pos: [350, 100],
      type: 2,
      size: [250, 300],
      flags: { bypass: false },
      order: 1,
      mode: 0,
      inputs: [
        { name: 'model', type: 'MODEL', link: null },
        { name: 'latent_image', type: 'LATENT', link: '1' },
      ],
      outputs: [{ name: 'LATENT', type: 'LATENT', links: null }],
      properties: {},
      widgets_values: [20, 7.5, 'euler', 'normal', 1],
    },
  ],
  links: [
    {
      id: '1',
      origin_id: '1',
      origin_slot: 0,
      target_id: '2',
      target_slot: 1,
      type: 'IMAGE',
    },
  ],
  widgets: [
    { id: 'w1', nodeId: '2', name: 'steps', type: 'number', value: 20 },
    { id: 'w2', nodeId: '2', name: 'cfg', type: 'number', value: 7.5 },
    { id: 'w3', nodeId: '2', name: 'sampler_name', type: 'combo', value: 'euler', options: ['euler', 'ddim', 'dpm++'] },
  ],
}

let store: ReturnType<typeof useSpywareStore<GraphData>>
let activeRefs = new Map<string, any>()
let pathAccessCounts = new Map<string, number>()
let proxyCreations: Array<{ path: string, timestamp: number }> = []
let performanceMetrics = reactive({
  totalPatches: 0,
  patchesPerSecond: 0,
  activeSubscriptions: 0,
  activeProxies: 0,
  lastPatchTime: 0,
})

const reactiveConnections = ref<Array<{
  from: string
  to: string
  active: boolean
  pulseTime?: number
}>>([])

const mountedComponents = ref<Array<{
  id: string
  name: string
  subscriptions: string[]
  mounted: boolean
  scope?: ReturnType<typeof effectScope>
}>>([])

const patches = ref<Patch[]>([])
const inversePatches = ref<Patch[]>([])

// Track created proxies
const proxyCache = new WeakMap()
const proxyPathsCreated = new Set<string>()

// Create instrumented proxy factory
function createInstrumentedProxy(obj: any, path: string): any {
  if (typeof obj !== 'object' || obj === null) return obj
  if (proxyCache.has(obj)) return proxyCache.get(obj)
  
  // Track unique paths
  if (!proxyPathsCreated.has(path)) {
    proxyPathsCreated.add(path)
    proxyCreations.push({ path: path || 'root', timestamp: Date.now() })
    performanceMetrics.activeProxies = proxyPathsCreated.size
  }
  
  const proxy = new Proxy(obj, {
    get(target, prop) {
      // Skip Vue internal properties
      if (typeof prop === 'symbol' || String(prop).startsWith('__')) return target[prop]
      
      const fullPath = path ? `${path}.${String(prop)}` : String(prop)
      pathAccessCounts.set(fullPath, (pathAccessCounts.get(fullPath) || 0) + 1)
      
      // Trigger reactivity visualization if connections exist
      if (reactiveConnections.value && Array.isArray(reactiveConnections.value)) {
        reactiveConnections.value.forEach(conn => {
          if (conn.from === fullPath) {
            conn.active = true
            setTimeout(() => { conn.active = false }, 300)
          }
        })
      }
      
      const value = target[prop]
      // Recursively create proxies for nested objects
      if (value && typeof value === 'object' && !Array.isArray(value) && typeof value !== 'function') {
        return createInstrumentedProxy(value, fullPath)
      }
      return value
    },
    set(target, prop, value) {
      target[prop] = value
      return true
    }
  })
  
  proxyCache.set(obj, proxy)
  return proxy
}

demoScope.run(() => {
  store = useSpywareStore(initialGraph)
  window.store = store
  
  // Instrument the store's state getter to track proxy creation
  const originalState = Object.getOwnPropertyDescriptor(store, 'state')!
  Object.defineProperty(store, 'state', {
    get() {
      const state = originalState.get!.call(this)
      return createInstrumentedProxy(state, 'state')
    },
    configurable: true
  })

  store.subscribe((p, ip) => {
    patches.value.push(...p)
    inversePatches.value.push(...ip)

    // Animate reactive connections
    p.forEach(patch => {
      const path = patch.path.join('.')
      const connections = reactiveConnections.value.filter(c => c.from === path)
      connections.forEach(c => {
        c.active = true
        c.pulseTime = Date.now()
        setTimeout(() => { c.active = false }, 500)
      })
    })
  })

  // Performance monitoring
  setInterval(() => {
    const now = Date.now()
    const recentPatches = patches.value.filter(() => {
      return now - performanceMetrics.lastPatchTime < 1000
    })
    performanceMetrics.patchesPerSecond = recentPatches.length
    performanceMetrics.activeSubscriptions = activeRefs.size
  }, 100)
})

onBeforeUnmount(() => {
  demoScope.stop()
})

onUnmounted(() => {
  window.spyware = spyware
  delete window.store
})

function createMountableComponent(name: string, paths: string[]) {
  const id = `comp-${Date.now()}-${Math.random()}`
  const component = {
    id,
    name,
    subscriptions: paths,
    mounted: false,
  }
  
  mountedComponents.value.push(component)
  return component
}

function mountComponent(component: any) {
  component.mounted = true
  component.scope = effectScope()
  
  component.scope.run(() => {
    component.subscriptions.forEach((path: string) => {
      const ref = store.$ref(path)
      activeRefs.set(`${component.id}-${path}`, { ref, scope: component.scope })
      
      reactiveConnections.value.push({
        from: path,
        to: component.name,
        active: false,
      })
    })
  })
  
  performanceMetrics.activeSubscriptions = activeRefs.size
}

function unmountComponent(component: any) {
  component.mounted = false
  
  if (component.scope) {
    component.scope.stop()
  }
  
  component.subscriptions.forEach((path: string) => {
    const key = `${component.id}-${path}`
    activeRefs.delete(key)
  })
  
  reactiveConnections.value = reactiveConnections.value.filter(
    c => c.to !== component.name
  )
  
  performanceMetrics.activeSubscriptions = activeRefs.size
}

const terminalOutput = computed(() => {
  const lines = []
  lines.push('╔══════════════════════════════════════════════════════════════╗')
  lines.push('║  COMFY SPYWARE v0.0.1  [REACTIVITY MONITOR]  120BPM         ║')
  lines.push('╚══════════════════════════════════════════════════════════════╝')
  lines.push('')
  lines.push(`PATCHES: ${patches.value.length} | RATE: ${performanceMetrics.patchesPerSecond}/s`)
  lines.push(`PROXIES: ${performanceMetrics.activeProxies} | SUBS: ${performanceMetrics.activeSubscriptions}`)
  lines.push('')
  
  if (patches.value.length > 0) {
    const lastPatch = patches.value[patches.value.length - 1]
    lines.push(`LAST: ${lastPatch.op} ${lastPatch.path.join('.')} = ${JSON.stringify(lastPatch.value)}`)
  }
  
  return lines.join('\n')
})

const proxyTreeVisualization = computed(() => {
  const tree: Record<string, Set<string>> = {}
  
  // Always show root
  tree['root'] = new Set()
  
  proxyCreations.forEach(({ path }) => {
    if (!path || path === 'root') {
      tree['root'].add('state')
      return
    }
    
    const parts = path.split('.')
    let current = 'root'
    
    parts.forEach((part, i) => {
      if (!tree[current]) tree[current] = new Set()
      const next = i === 0 ? part : parts.slice(0, i + 1).join('.')
      tree[current].add(next)
      current = next
    })
  })
  
  return tree
})

// Pre-create and mount example components
const exampleComponents = [
  createMountableComponent('NodeEditor', ['nodes', 'nodes.0.title']),
  createMountableComponent('LinkRenderer', ['links']),
  createMountableComponent('WidgetPanel', ['widgets', 'nodes.0.properties']),
]

// Mount components by default
onMounted(() => {
  exampleComponents.forEach(comp => mountComponent(comp))
})

// Helper to get node position
function getNodePosition(nodeId: string) {
  const node = store.state.nodes.find(n => n.id === nodeId)
  return node ? { x: node.pos[0], y: node.pos[1] } : { x: 0, y: 0 }
}
</script>

<template>
  <div class="comfy-terminal-container">
    <div class="terminal-glow" />
    
    <header class="terminal-header">
      <h1 class="glitch" data-text="SPYWARE">SPYWARE</h1>
      <div class="status-bar">
        <span class="status-live">● LIVE</span>
        <span>{{ new Date().toLocaleTimeString() }}</span>
        <span>FPS: 60</span>
      </div>
    </header>

    <div class="dashboard-grid">
      <!-- Graph Visualization -->
      <section class="panel graph-editor">
        <h2 class="panel-title">◢ NODE GRAPH ◣</h2>
        <div class="graph-canvas">
          <svg viewBox="0 0 800 400" class="node-graph-svg">
            <!-- Links -->
            <g class="links-layer">
              <path
                v-for="link in store.state.links"
                :key="link.id"
                :d="`M ${getNodePosition(link.origin_id).x + 150} ${getNodePosition(link.origin_id).y + 50} 
                     L ${getNodePosition(link.target_id).x} ${getNodePosition(link.target_id).y + 50}`"
                class="graph-link"
                :stroke="link.type === 'IMAGE' ? '#F0FF41' : '#172DD7'"
              />
            </g>
            
            <!-- Nodes -->
            <g class="nodes-layer">
              <g v-for="node in store.state.nodes" :key="node.id" 
                 :transform="`translate(${node.pos[0]}, ${node.pos[1]})`"
                 class="graph-node-group">
                <rect 
                  :width="node.size[0]" 
                  :height="node.size[1]"
                  class="graph-node"
                  :class="{ bypass: node.flags.bypass, mute: node.flags.mute }"
                />
                <text x="10" y="25" class="node-title">{{ node.title }}</text>
                <text x="10" y="45" class="node-type">Type: {{ node.type }}</text>
                
                <!-- Inputs/Outputs -->
                <g v-if="node.outputs">
                  <circle
                    v-for="(output, idx) in node.outputs"
                    :key="idx"
                    :cx="node.size[0]"
                    :cy="30 + idx * 20"
                    r="6"
                    class="node-port output-port"
                  />
                </g>
                <g v-if="node.inputs">
                  <circle
                    v-for="(input, idx) in node.inputs"
                    :key="idx"
                    cx="0"
                    :cy="30 + idx * 20"
                    r="6"
                    class="node-port input-port"
                  />
                </g>
              </g>
            </g>
          </svg>
          
          <div class="graph-controls">
            <div v-for="node in store.state.nodes" :key="node.id" class="node-control">
              <h4>{{ node.title }}</h4>
              <input 
                v-model="node.title" 
                class="node-title-input"
                @input="() => reactiveConnections.push({ from: `nodes.${store.state.nodes.indexOf(node)}.title`, to: 'UI', active: true, pulseTime: Date.now() })"
              />
              <label>
                <input type="checkbox" v-model="node.flags.bypass" />
                <span>Bypass</span>
              </label>
              <label>
                <input type="checkbox" v-model="node.flags.mute" />
                <span>Mute</span>
              </label>
            </div>
          </div>
        </div>
      </section>

      <!-- Reactivity Visualization -->
      <section class="panel reactivity-viz">
        <h2 class="panel-title">◢ REACTIVITY FLOW ◣</h2>
        <div class="reactivity-canvas">
          <svg viewBox="0 0 400 300" class="flow-svg">
            <defs>
              <filter id="glow">
                <feGaussianBlur stdDeviation="3" result="coloredBlur"/>
                <feMerge>
                  <feMergeNode in="coloredBlur"/>
                  <feMergeNode in="SourceGraphic"/>
                </feMerge>
              </filter>
            </defs>
            
            <g v-for="(conn, idx) in reactiveConnections.slice(0, 10)" :key="`${conn.from}-${conn.to}-${idx}`">
              <line
                :x1="100"
                :y1="50 + idx * 20"
                :x2="300"
                :y2="150"
                :class="['connection', { active: conn.active }]"
                stroke-width="2"
              />
              <circle
                v-if="conn.active"
                :cx="100"
                :cy="50 + idx * 20"
                r="3"
                class="pulse-dot"
              >
                <animate
                  attributeName="cx"
                  :from="100"
                  :to="300"
                  dur="0.5s"
                  repeatCount="1"
                />
              </circle>
            </g>
            
            <text x="50" y="30" class="label">SOURCES</text>
            <text x="280" y="30" class="label">TARGETS</text>
          </svg>
          
          <div class="access-counter">
            <h3>PATH ACCESS COUNT</h3>
            <div v-for="[path, count] in Array.from(pathAccessCounts.entries()).sort((a, b) => b[1] - a[1]).slice(0, 5)" :key="`${path}-${count}`" class="access-item">
              <span class="path">{{ path }}</span>
              <span class="count">{{ count }}x</span>
            </div>
            <div v-if="pathAccessCounts.size === 0" class="no-access">
              No paths accessed yet
            </div>
            <div class="subscription-info">
              <h4>ACTIVE REFS: {{ performanceMetrics.activeSubscriptions }}</h4>
              <div v-if="performanceMetrics.activeSubscriptions === 0" class="cleanup-success">
                ✓ All component refs cleaned up!
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- Lazy Proxy Tree -->
      <section class="panel proxy-tree">
        <h2 class="panel-title">◢ LAZY PROXY TREE ◣</h2>
        <div class="tree-viz">
          <div class="tree-node" v-for="[parent, children] in Object.entries(proxyTreeVisualization).slice(0, 5)" :key="parent">
            <div class="tree-parent">{{ parent === 'root' ? '◈ ROOT' : parent }}</div>
            <div class="tree-children">
              <div v-for="child in Array.from(children).slice(0, 5)" :key="child" class="tree-child">
                ├─ {{ child.split('.').pop() }}
              </div>
            </div>
          </div>
          <div class="proxy-stats">
            TOTAL PROXIES: {{ performanceMetrics.activeProxies }}
            <div class="creation-log">
              <div v-for="(p, i) in proxyCreations.slice(-5).reverse()" :key="`${p.path}-${i}`" class="proxy-creation">
                {{ p.path || 'root' }}
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- Component Lifecycle Demo -->
      <section class="panel mount-demo">
        <h2 class="panel-title">◢ COMPONENT LIFECYCLE ◣</h2>
        <div class="component-list">
          <div v-for="comp in exampleComponents" :key="comp.id" class="component-item">
            <div class="comp-header">
              <span class="comp-name">{{ comp.name }}</span>
              <button 
                @click="comp.mounted ? unmountComponent(comp) : mountComponent(comp)"
                :class="['mount-btn', { mounted: comp.mounted }]"
              >
                {{ comp.mounted ? 'UNMOUNT' : 'MOUNT' }}
              </button>
            </div>
            <div v-if="comp.mounted" class="comp-subs">
              <div v-for="sub in comp.subscriptions" :key="sub" class="sub-item">
                ◦ {{ sub }}
              </div>
            </div>
          </div>
        </div>
        <div class="cleanup-indicator">
          AUTO-CLEANUP: <span class="status-on">ENABLED</span>
        </div>
      </section>

      <!-- Performance Metrics -->
      <section class="panel performance">
        <h2 class="panel-title">◢ PERFORMANCE METRICS ◣</h2>
        <div class="metrics-grid">
          <div class="metric">
            <div class="metric-value">{{ performanceMetrics.totalPatches }}</div>
            <div class="metric-label">TOTAL PATCHES</div>
          </div>
          <div class="metric">
            <div class="metric-value">{{ performanceMetrics.patchesPerSecond }}</div>
            <div class="metric-label">PATCHES/SEC</div>
          </div>
          <div class="metric">
            <div class="metric-value">{{ performanceMetrics.activeSubscriptions }}</div>
            <div class="metric-label">ACTIVE SUBS</div>
          </div>
          <div class="metric">
            <div class="metric-value">{{ performanceMetrics.activeProxies }}</div>
            <div class="metric-label">PROXY COUNT</div>
          </div>
        </div>
        <pre class="terminal-output">{{ terminalOutput }}</pre>
      </section>

      <!-- Patch History -->
      <section class="panel patch-history">
        <h2 class="panel-title">◢ PATCH STREAM ◣</h2>
        <div class="patch-list">
          <div 
            v-for="(patch, i) in patches.slice(-10).reverse()" 
            :key="i"
            class="patch-item"
            :class="{ latest: i === 0 }"
          >
            <span class="patch-op">{{ patch.op }}</span>
            <span class="patch-path">{{ patch.path.join('.') }}</span>
            <span class="patch-value">{{ JSON.stringify(patch.value) }}</span>
          </div>
        </div>
      </section>

      <!-- Widget Controls -->
      <section class="panel widget-panel">
        <h2 class="panel-title">◢ WIDGET CONTROLS ◣</h2>
        <div class="widget-list">
          <div v-for="widget in store.state.widgets" :key="widget.id" class="widget-item">
            <span class="widget-name">{{ widget.name }}</span>
            <input 
              v-if="widget.type === 'number'" 
              type="number" 
              v-model.number="widget.value"
              class="widget-input"
            />
            <select 
              v-else-if="widget.type === 'combo'" 
              v-model="widget.value"
              class="widget-select"
            >
              <option v-for="opt in widget.options" :key="opt" :value="opt">
                {{ opt }}
              </option>
            </select>
          </div>
        </div>
      </section>
    </div>

    <div class="console-hint">
      <span>▸</span> Open console and use: window.store.state
    </div>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Share+Tech+Mono&display=swap');

.comfy-terminal-container {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: #FFFFFF;
  color: #000;
  font-family: 'Share Tech Mono', monospace;
  overflow: auto;
}

/* Clean white background */

/* Remove scan line effect */

.terminal-glow {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: radial-gradient(circle at center, transparent 0%, rgba(23, 45, 215, 0.05) 100%);
  pointer-events: none;
  z-index: 1;
}

.terminal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  border-bottom: 3px solid #172DD7;
  background: #FFFFFF;
}

.glitch {
  font-size: 3rem;
  font-weight: 900;
  text-transform: uppercase;
  position: relative;
  color: #172DD7;
  letter-spacing: 0.2em;
}

/* Clean typography without glitch effects */

.status-bar {
  display: flex;
  gap: 2rem;
  font-size: 0.9rem;
  text-transform: uppercase;
}

.status-live {
  color: #F0FF41;
  font-weight: bold;
}

.dashboard-grid {
  display: grid;
  grid-template-columns: 2fr 1fr 1fr;
  grid-template-rows: 2fr 2fr 1fr;
  gap: 1rem;
  padding: 1rem;
  height: calc(100vh - 100px);
  overflow: hidden;
}

.panel {
  background: #FFFFFF;
  border: 3px solid #172DD7;
  padding: 1rem;
  position: relative;
  overflow: hidden;
  box-shadow: 5px 5px 0 #F0FF41;
}

.panel::before {
  content: '';
  position: absolute;
  top: -2px;
  left: -2px;
  right: -2px;
  bottom: -2px;
  background: linear-gradient(45deg, #F0FF41, #172DD7, #F0FF41);
  opacity: 0;
  z-index: -1;
  transition: opacity 0.3s;
}

.panel:hover {
  box-shadow: 8px 8px 0 #F0FF41;
  transform: translate(-2px, -2px);
}

.panel-title {
  font-size: 1.2rem;
  margin-bottom: 1rem;
  color: #172DD7;
  font-weight: 900;
  letter-spacing: 0.1em;
}

.graph-editor {
  grid-column: 1;
  grid-row: 1 / 3;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.reactivity-viz {
  grid-column: 2;
  grid-row: 1;
  overflow: hidden;
}

.proxy-tree {
  grid-column: 3;
  grid-row: 1;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.mount-demo {
  grid-column: 2;
  grid-row: 2;
  overflow: hidden;
}

.performance {
  grid-column: 3;
  grid-row: 2;
  overflow: hidden;
}

.tree-viz {
  flex: 1;
  overflow-y: auto;
}

.patch-history {
  grid-column: 1 / 3;
  grid-row: 3;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.widget-panel {
  grid-column: 3;
  grid-row: 3;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.patch-list {
  flex: 1;
  overflow-y: auto;
  padding-right: 0.5rem;
}

.node-item {
  background: #F0FF41;
  border: 2px solid #172DD7;
  padding: 0.5rem;
  margin-bottom: 0.5rem;
  position: relative;
}

.node-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
}

.node-id {
  font-size: 0.8rem;
  color: #172DD7;
}

.node-title-input {
  background: #FFFFFF;
  border: 2px solid #172DD7;
  color: #000;
  padding: 0.2rem 0.5rem;
  font-family: inherit;
  font-weight: 600;
}

.node-props label {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  margin-right: 1rem;
  cursor: pointer;
}

input[type="checkbox"] {
  appearance: none;
  width: 1rem;
  height: 1rem;
  border: 2px solid #172DD7;
  background: #FFF;
  position: relative;
  cursor: pointer;
}

input[type="checkbox"]:checked {
  background: #F0FF41;
}

input[type="checkbox"]:checked::after {
  content: '✓';
  position: absolute;
  top: -0.1rem;
  left: 0.2rem;
  color: #172DD7;
  font-size: 0.8rem;
  font-weight: 900;
}

.widget-list {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.widget-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.5rem;
  background: #F0FF41;
  border: 2px solid #172DD7;
}

.widget-name {
  text-transform: uppercase;
  font-size: 0.9rem;
}

.widget-input,
.widget-select {
  background: #FFF;
  border: 2px solid #172DD7;
  color: #000;
  padding: 0.2rem 0.5rem;
  font-family: inherit;
  font-weight: 600;
}

.flow-svg {
  width: 100%;
  height: 200px;
}

.connection {
  stroke: #172DD7;
  opacity: 0.2;
  stroke-width: 3;
  transition: all 0.3s;
}

.connection.active {
  stroke: #F0FF41;
  opacity: 1;
  stroke-width: 5;
}

.pulse-dot {
  fill: #F0FF41;
}

.label {
  fill: #172DD7;
  font-size: 12px;
  text-transform: uppercase;
  font-weight: 900;
}

.access-counter {
  margin-top: 1rem;
  padding: 0.5rem;
  background: #F0FF41;
  border: 2px solid #172DD7;
}

.access-item {
  display: flex;
  justify-content: space-between;
  font-size: 0.8rem;
  margin-bottom: 0.2rem;
}

.path {
  color: #000;
  font-weight: 600;
}

.count {
  color: #172DD7;
  font-weight: 900;
}

.tree-node {
  margin-bottom: 0.5rem;
}

.tree-parent {
  color: #172DD7;
  margin-bottom: 0.2rem;
  font-weight: 900;
}

.tree-child {
  color: #000;
  padding-left: 1rem;
  font-size: 0.9rem;
  font-weight: 600;
}

.proxy-stats {
  margin-top: 1rem;
  padding: 0.5rem;
  background: #172DD7;
  color: #F0FF41;
  text-align: center;
  font-weight: 900;
}

.creation-log {
  margin-top: 0.5rem;
  font-size: 0.8rem;
  text-align: left;
}

.proxy-creation {
  padding: 0.2rem;
  border-bottom: 1px solid rgba(240, 255, 65, 0.3);
}

.no-access {
  text-align: center;
  color: #999;
  padding: 1rem;
  font-style: italic;
}

.subscription-info {
  margin-top: 1rem;
  padding: 0.5rem;
  background: #FFF;
  border: 2px solid #172DD7;
}

.subscription-info h4 {
  margin: 0;
  color: #172DD7;
  font-size: 0.9rem;
}

.cleanup-success {
  color: #08501A;
  font-weight: 900;
  margin-top: 0.5rem;
}

.mini-chart {
  display: flex;
  align-items: flex-end;
  height: 40px;
  gap: 2px;
  margin-top: 0.5rem;
}

.proxy-bar {
  flex: 1;
  background: #172DD7;
  min-height: 5px;
}

.component-item {
  background: #F0FF41;
  border: 2px solid #172DD7;
  padding: 0.5rem;
  margin-bottom: 0.5rem;
}

.comp-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.mount-btn {
  background: #FFF;
  border: 2px solid #172DD7;
  color: #172DD7;
  padding: 0.2rem 0.5rem;
  cursor: pointer;
  text-transform: uppercase;
  font-family: inherit;
  font-weight: 900;
  transition: all 0.3s;
}

.mount-btn:hover {
  background: #172DD7;
  color: #F0FF41;
}

.mount-btn.mounted {
  background: #172DD7;
  color: #F0FF41;
}

.comp-subs {
  margin-top: 0.5rem;
  padding-left: 1rem;
  font-size: 0.8rem;
  color: #000;
  font-weight: 600;
}

.cleanup-indicator {
  margin-top: 1rem;
  text-align: center;
  padding: 0.5rem;
  background: #172DD7;
  color: #FFF;
}

.status-on {
  color: #F0FF41;
  font-weight: 900;
}

.metrics-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 1rem;
  margin-bottom: 1rem;
}

.metric {
  text-align: center;
  padding: 0.5rem;
  background: #F0FF41;
  border: 2px solid #172DD7;
}

.metric-value {
  font-size: 2rem;
  color: #172DD7;
  font-weight: 900;
}

.metric-label {
  font-size: 0.8rem;
  color: #000;
  text-transform: uppercase;
  font-weight: 600;
}

.terminal-output {
  background: #172DD7;
  padding: 0.5rem;
  font-size: 0.8rem;
  color: #F0FF41;
  border: 2px solid #172DD7;
  white-space: pre;
  overflow: auto;
  font-weight: 600;
}

.patch-item {
  display: flex;
  gap: 1rem;
  padding: 0.5rem;
  font-size: 0.8rem;
  background: #FFF;
  border: 2px solid #172DD7;
  margin-bottom: 0.3rem;
  transition: all 0.3s;
}

.patch-item.latest {
  background: #F0FF41;
}

.patch-op {
  color: #172DD7;
  text-transform: uppercase;
  min-width: 60px;
  font-weight: 900;
}

.patch-path {
  color: #000;
  flex: 1;
  font-weight: 600;
}

.patch-value {
  color: #172DD7;
  font-weight: 600;
}

.console-hint {
  position: fixed;
  bottom: 1rem;
  left: 50%;
  transform: translateX(-50%);
  background: #172DD7;
  color: #F0FF41;
  padding: 0.5rem 1rem;
  border: 3px solid #F0FF41;
  font-size: 0.9rem;
  z-index: 10;
  font-weight: 900;
}

/* Node Graph Styles */
.graph-canvas {
  height: 100%;
  display: flex;
  flex-direction: column;
}

.node-graph-svg {
  flex: 1;
  background: #FFF;
  border: 2px solid #172DD7;
}

.graph-link {
  stroke-width: 3;
  fill: none;
}

.graph-node {
  fill: #F0FF41;
  stroke: #172DD7;
  stroke-width: 3;
}

.graph-node.bypass {
  opacity: 0.5;
}

.graph-node.mute {
  fill: #DDD;
}

.node-title {
  font-weight: 900;
  fill: #172DD7;
}

.node-type {
  font-size: 12px;
  fill: #000;
}

.node-port {
  stroke: #172DD7;
  stroke-width: 2;
}

.output-port {
  fill: #F0FF41;
}

.input-port {
  fill: #FFF;
}

.graph-controls {
  display: flex;
  gap: 1rem;
  padding: 1rem 0;
  overflow-x: auto;
}

.node-control {
  background: #F0FF41;
  border: 2px solid #172DD7;
  padding: 0.5rem;
  min-width: 200px;
}

.node-control h4 {
  margin: 0 0 0.5rem 0;
  color: #172DD7;
  font-weight: 900;
}

@media (max-width: 768px) {
  .dashboard-grid {
    grid-template-columns: 1fr;
  }
  
  .graph-editor {
    grid-column: span 1;
  }
}
</style>