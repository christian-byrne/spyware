<script setup lang="ts">
import type { Patch } from 'immer'
import { computed, ref } from 'vue'
import { patchState } from '~/composables/spyware'
import { useSpywareStore } from '~/composables/vueSpyware'

// Type definitions
interface Widget {
  name: string
  value: any
}

interface GraphNode {
  title: string
  type: string
  bypass: boolean
  mute: boolean
  cached: boolean
  widgets: Widget[]
}

interface GraphLink {
  source: string
  target: string
}

interface GraphData {
  nodes: Record<string, GraphNode>
  links: GraphLink[]
}

// Initialize store with graph data
const store = useSpywareStore<GraphData>({
  nodes: {
    node_1: {
      title: 'Load Image',
      type: 'LoadImage',
      bypass: false,
      mute: false,
      cached: true,
      widgets: [
        { name: 'image', value: 'example.png' },
      ],
    },
    node_2: {
      title: 'VAE Encode',
      type: 'VAEEncode',
      bypass: false,
      mute: false,
      cached: false,
      widgets: [],
    },
    node_3: {
      title: 'KSampler',
      type: 'KSampler',
      bypass: false,
      mute: true,
      cached: false,
      widgets: [
        { name: 'seed', value: 42 },
        { name: 'steps', value: 20 },
      ],
    },
  },
  links: [
    { source: 'node_1', target: 'node_2' },
    { source: 'node_2', target: 'node_3' },
  ],
})

// Create reactive refs
const nodes = store.$ref<Record<string, GraphNode>>('nodes')
const links = store.$ref<GraphLink[]>('links')

// Node ID counter
let nodeCounter = 4

// Patch history tracking
interface PatchEntry {
  patches: Patch[]
  inversePatches: Patch[]
  timestamp: string
}

const patchHistory = ref<PatchEntry[]>([])

// Subscribe to all changes
store.subscribe((patches, inversePatches) => {
  console.log('📝 Patches received:', patches)
  console.log('↩️  Inverse patches:', inversePatches)

  if (patches.length > 0) {
    patchHistory.value.unshift({
      patches,
      inversePatches,
      timestamp: new Date().toLocaleTimeString(),
    })

    // Keep only last 50 entries
    if (patchHistory.value.length > 50) {
      patchHistory.value = patchHistory.value.slice(0, 50)
    }
  }
})

// Computed formatted state
const formattedState = computed(() => {
  return JSON.stringify(store.state, null, 2)
})

// Methods
function addNode() {
  const nodeId = `node_${nodeCounter++}`
  nodes.value[nodeId] = {
    title: 'New Node',
    type: 'Unknown',
    bypass: false,
    mute: false,
    cached: false,
    widgets: [],
  }
}

function deleteNode(nodeId: string) {
  // Remove node
  delete nodes.value[nodeId]

  // Remove any links connected to this node
  links.value = links.value.filter(link =>
    link.source !== nodeId && link.target !== nodeId,
  )
}

function addWidget(nodeId: string) {
  nodes.value[nodeId].widgets.push({
    name: '',
    value: '',
  })
}

function removeWidget(nodeId: string, widgetIdx: number) {
  nodes.value[nodeId].widgets.splice(widgetIdx, 1)
}

function addLink() {
  links.value.push({
    source: '',
    target: '',
  })
}

function removeLink(linkIdx: number) {
  links.value.splice(linkIdx, 1)
}

function clearHistory() {
  patchHistory.value = []
}

function commitChanges() {
  console.log('💾 Committing changes...')
  console.log('  Before commit - isDirty:', store.spyware.isDirty)
  store.spyware.commit()
  console.log('  After commit - isDirty:', store.spyware.isDirty)
}

function undoChange(patches: Patch[]) {
  console.log('⏪ Applying undo patches:', patches)
  console.log('  Current state before undo:', JSON.parse(JSON.stringify(store.state)))
  patchState(store.spyware, patches)
  console.log('  State after undo:', JSON.parse(JSON.stringify(store.state)))
}

// Expose store and utilities for console access
if (typeof window !== 'undefined') {
  (window as any).graphStore = store
  ;(window as any).patchState = patchState
  console.log('🎮 Graph store ready! Access via:')
  console.log('  - graphStore: The Vue-integrated spyware store')
  console.log('  - patchState: Function to apply patches')
  console.log('📝 Check the "Console Snippets" section for examples!')
}
</script>

<template>
  <div class="spyware-demo mx-auto p-6 max-w-6xl">
    <h1 class="text-3xl font-bold mb-6">
      Vue Spyware Integration Demo - Graph Editor
    </h1>

    <div class="gap-6 grid grid-cols-1 lg:grid-cols-2">
      <!-- Graph State Editor -->
      <div class="card p-4 rounded-lg bg-gray-100 dark:bg-gray-800">
        <h2 class="text-xl font-semibold mb-4">
          Edit Graph
        </h2>

        <!-- Node Editor -->
        <div class="space-y-4">
          <h3 class="font-semibold">
            Nodes
          </h3>
          <div v-for="(node, nodeId) in nodes" :key="nodeId" class="p-3 border rounded dark:border-gray-600">
            <div class="mb-2 flex items-center justify-between">
              <span class="text-sm font-mono">{{ nodeId }}</span>
              <button
                class="text-sm text-red-500 hover:text-red-600"
                @click="deleteNode(nodeId)"
              >
                Delete
              </button>
            </div>

            <div class="text-sm gap-2 grid grid-cols-2">
              <input
                v-model="node.title"
                placeholder="Node title"
                class="px-2 py-1 border rounded dark:border-gray-600 dark:bg-gray-700"
              >
              <input
                v-model="node.type"
                placeholder="Node type"
                class="px-2 py-1 border rounded dark:border-gray-600 dark:bg-gray-700"
              >
            </div>

            <div class="mt-2 flex gap-2">
              <label class="text-sm flex gap-1 items-center">
                <input v-model="node.bypass" type="checkbox">
                Bypass
              </label>
              <label class="text-sm flex gap-1 items-center">
                <input v-model="node.mute" type="checkbox">
                Mute
              </label>
              <label class="text-sm flex gap-1 items-center">
                <input v-model="node.cached" type="checkbox">
                Cached
              </label>
            </div>

            <!-- Widgets -->
            <div class="mt-2">
              <div class="mb-1 flex items-center justify-between">
                <span class="text-xs font-semibold">Widgets</span>
                <button
                  class="text-xs text-blue-500 hover:text-blue-600"
                  @click="addWidget(nodeId)"
                >
                  + Add
                </button>
              </div>
              <div v-for="(widget, widgetIdx) in node.widgets" :key="widgetIdx" class="mb-1 flex gap-1">
                <input
                  v-model="widget.name"
                  placeholder="Widget name"
                  class="text-xs px-1 py-0.5 border rounded flex-1 dark:border-gray-600 dark:bg-gray-700"
                >
                <input
                  v-model="widget.value"
                  placeholder="Value"
                  class="text-xs px-1 py-0.5 border rounded w-20 dark:border-gray-600 dark:bg-gray-700"
                >
                <button
                  class="text-xs text-red-500 hover:text-red-600"
                  @click="removeWidget(nodeId, widgetIdx)"
                >
                  ×
                </button>
              </div>
            </div>
          </div>

          <button
            class="text-white px-3 py-2 rounded bg-green-500 w-full hover:bg-green-600"
            @click="addNode"
          >
            + Add Node
          </button>
        </div>

        <!-- Link Editor -->
        <div class="mt-6 space-y-4">
          <h3 class="font-semibold">
            Links
          </h3>
          <div v-for="(link, linkIdx) in links" :key="linkIdx" class="flex gap-2 items-center">
            <select
              v-model="link.source"
              class="text-sm px-2 py-1 border rounded flex-1 dark:border-gray-600 dark:bg-gray-700"
            >
              <option value="">
                Source Node
              </option>
              <option v-for="nodeId in Object.keys(nodes)" :key="nodeId" :value="nodeId">
                {{ nodeId }} - {{ nodes[nodeId].title }}
              </option>
            </select>
            <span>→</span>
            <select
              v-model="link.target"
              class="text-sm px-2 py-1 border rounded flex-1 dark:border-gray-600 dark:bg-gray-700"
            >
              <option value="">
                Target Node
              </option>
              <option v-for="nodeId in Object.keys(nodes)" :key="nodeId" :value="nodeId">
                {{ nodeId }} - {{ nodes[nodeId].title }}
              </option>
            </select>
            <button
              class="text-red-500 hover:text-red-600"
              @click="removeLink(linkIdx)"
            >
              ×
            </button>
          </div>

          <button
            class="text-white px-3 py-2 rounded bg-blue-500 w-full hover:bg-blue-600"
            @click="addLink"
          >
            + Add Link
          </button>
        </div>
      </div>

      <!-- State Display -->
      <div class="card p-4 rounded-lg bg-gray-100 dark:bg-gray-800">
        <h2 class="text-xl font-semibold mb-4">
          Current Graph State
        </h2>
        <pre class="text-sm p-3 rounded bg-gray-200 max-h-96 overflow-auto dark:bg-gray-900">{{ formattedState }}</pre>

        <div class="mt-4 space-y-2">
          <p class="text-sm">
            <span class="font-mono">isDirty:</span>
            <span :class="store.spyware.isDirty ? 'text-yellow-500' : 'text-green-500'">
              {{ store.spyware.isDirty }}
            </span>
          </p>
          <p class="text-sm">
            <span class="font-mono">Nodes:</span> {{ Object.keys(nodes).length }}
          </p>
          <p class="text-sm">
            <span class="font-mono">Links:</span> {{ links.length }}
          </p>
        </div>

        <div class="mt-4">
          <button
            class="text-white px-4 py-2 rounded bg-green-500 hover:bg-green-600"
            :disabled="!store.spyware.isDirty"
            :class="{ 'opacity-50 cursor-not-allowed': !store.spyware.isDirty }"
            @click="commitChanges"
          >
            Commit Changes
          </button>
        </div>
      </div>
    </div>

    <!-- Patch History -->
    <div class="card mt-6 p-4 rounded-lg bg-gray-100 dark:bg-gray-800">
      <div class="mb-4 flex items-center justify-between">
        <h2 class="text-xl font-semibold">
          Patch History
        </h2>
        <button
          class="text-sm text-white px-3 py-1 rounded bg-red-500 hover:bg-red-600"
          @click="clearHistory"
        >
          Clear History
        </button>
      </div>

      <div class="max-h-96 overflow-auto space-y-2">
        <div
          v-for="(entry, index) in patchHistory"
          :key="`${index}-${entry.timestamp}`"
          class="p-3 border border-gray-300 rounded dark:border-gray-600"
        >
          <div class="mb-2 flex items-start justify-between">
            <span class="text-sm font-semibold">Change #{{ patchHistory.length - index }}</span>
            <span class="text-xs text-gray-500">{{ entry.timestamp }}</span>
          </div>

          <div class="text-sm gap-2 grid grid-cols-1 md:grid-cols-2">
            <div>
              <span class="font-medium">Forward Patches:</span>
              <pre class="text-xs mt-1 p-2 rounded bg-gray-200 overflow-auto dark:bg-gray-900">{{ JSON.stringify(entry.patches, null, 2) }}</pre>
            </div>
            <div>
              <span class="font-medium">Inverse Patches (Undo):</span>
              <pre class="text-xs mt-1 p-2 rounded bg-gray-200 overflow-auto dark:bg-gray-900">{{ JSON.stringify(entry.inversePatches, null, 2) }}</pre>
            </div>
          </div>

          <button
            class="text-sm text-white mt-2 px-3 py-1 rounded bg-yellow-500 hover:bg-yellow-600"
            @click="undoChange(entry.inversePatches)"
          >
            Undo This Change
          </button>
        </div>

        <div v-if="patchHistory.length === 0" class="text-gray-500 py-8 text-center">
          No changes recorded yet. Try editing the graph above!
        </div>
      </div>
    </div>

    <!-- Console Snippets -->
    <div class="card mt-6 p-4 rounded-lg bg-gray-100 dark:bg-gray-800">
      <h2 class="text-xl font-semibold mb-4">
        🛠️ Console Snippets - Test Reactivity
      </h2>
      <p class="text-sm text-gray-600 mb-4 dark:text-gray-400">
        Copy and paste these snippets in your DevTools console to visualize reactivity:
      </p>

      <div class="space-y-4">
        <div>
          <h3 class="text-sm font-semibold mb-2">
            1. Basic Reactivity Test
          </h3>
          <pre class="text-xs text-green-400 p-3 rounded bg-gray-900 overflow-x-auto">// Watch a specific node's title
const titleRef = graphStore.$ref('nodes.node_1.title')
console.log('Current title:', titleRef.value)

// Change it and see patches
titleRef.value = 'Updated from Console!'</pre>
        </div>

        <div>
          <h3 class="text-sm font-semibold mb-2">
            2. Subscribe to Node Changes
          </h3>
          <pre class="text-xs text-green-400 p-3 rounded bg-gray-900 overflow-x-auto">// Subscribe to all node changes
const unsubscribe = graphStore.subscribePath('nodes', (patches) => {
  console.log('🔔 Nodes changed!', patches.map(p => ({
    path: p.path.join('.'),
    op: p.op,
    value: p.value
  })))
})

// Test it - then call unsubscribe() when done
graphStore.state.nodes.node_1.mute = !graphStore.state.nodes.node_1.mute</pre>
        </div>

        <div>
          <h3 class="text-sm font-semibold mb-2">
            3. Batch Updates & Reactivity
          </h3>
          <pre class="text-xs text-green-400 p-3 rounded bg-gray-900 overflow-x-auto">// Multiple changes in one tick
console.log('🚀 Starting batch update...')
graphStore.state.nodes.node_2.bypass = true
graphStore.state.nodes.node_2.title = 'Batch Updated'
graphStore.state.nodes.node_2.widgets = [{name: 'test', value: 123}]
console.log('✅ Batch complete - check patches above!')</pre>
        </div>

        <div>
          <h3 class="text-sm font-semibold mb-2">
            4. Create Reactive Effects
          </h3>
          <pre class="text-xs text-green-400 p-3 rounded bg-gray-900 overflow-x-auto">// Create multiple reactive refs
const refs = graphStore.toRefs({
  node1Title: 'nodes.node_1.title',
  node1Mute: 'nodes.node_1.mute',
  linkCount: 'links'
})

// Log whenever they're accessed
Object.entries(refs).forEach(([key, ref]) => {
  console.log(`${key}:`, ref.value)
})</pre>
        </div>

        <div>
          <h3 class="text-sm font-semibold mb-2">
            5. Test Undo/Redo
          </h3>
          <pre class="text-xs text-green-400 p-3 rounded bg-gray-900 overflow-x-auto">// Capture changes for manual undo
let lastInverse = null
const capture = graphStore.subscribe((p, inverse) => {
  lastInverse = inverse
  console.log('📸 Captured inverse patches:', inverse)
})

// Make a change
graphStore.state.nodes.node_3.title = 'Will be undone'

// Undo it!
if (lastInverse) {
  patchState(graphStore.spyware, lastInverse)
  console.log('⏪ Undone!')
}</pre>
        </div>

        <div>
          <h3 class="text-sm font-semibold mb-2">
            6. Debug Current State
          </h3>
          <pre class="text-xs text-green-400 p-3 rounded bg-gray-900 overflow-x-auto">// Inspect store internals
console.log('📊 Store state:', graphStore.state)
console.log('🚦 Is dirty?', graphStore.spyware.isDirty)
console.log('🔗 Node count:', Object.keys(graphStore.state.nodes).length)
console.log('🔗 Link count:', graphStore.state.links.length)

// Test dirty flag
graphStore.state.nodes.node_1.cached = !graphStore.state.nodes.node_1.cached
console.log('After change - isDirty:', graphStore.spyware.isDirty)</pre>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.card {
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.dark .card {
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
}
</style>
