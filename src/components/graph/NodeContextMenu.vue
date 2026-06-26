<template>
  <DropdownMenu v-model:open="isOpen" :modal="false">
    <DropdownMenuTrigger as-child>
      <button
        type="button"
        aria-hidden="true"
        tabindex="-1"
        class="pointer-events-none fixed size-0 opacity-0"
        :style="anchorStyle"
      />
    </DropdownMenuTrigger>
    <DropdownMenuContent
      size="lg"
      align="start"
      :side-offset="0"
      :collision-padding="8"
      update-position-strategy="always"
      class="max-h-[80vh] overflow-y-auto md:max-h-none md:overflow-y-visible"
      @focus-outside="onFocusOutside"
    >
      <NodeContextMenuItem
        v-for="(option, idx) in menuOptions"
        :key="optionKey(option, idx)"
        :option
        @select="runAction"
        @submenu-select="handleSubmenuSelect"
      />
    </DropdownMenuContent>
  </DropdownMenu>
</template>

<script setup lang="ts">
import { useElementBounding, useRafFn } from '@vueuse/core'
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'

import DropdownMenu from '@/components/ui/dropdown-menu/DropdownMenu.vue'
import DropdownMenuContent from '@/components/ui/dropdown-menu/DropdownMenuContent.vue'
import DropdownMenuTrigger from '@/components/ui/dropdown-menu/DropdownMenuTrigger.vue'
import {
  registerNodeOptionsInstance,
  useMoreOptionsMenu
} from '@/composables/graph/useMoreOptionsMenu'
import type {
  MenuOption,
  SubMenuOption
} from '@/composables/graph/useMoreOptionsMenu'
import { useCanvasStore } from '@/renderer/core/canvas/canvasStore'

import NodeContextMenuItem from './NodeContextMenuItem.vue'

const OPEN_FOCUS_GRACE_MS = 300

const isOpen = ref(false)
const worldPosition = ref({ x: 0, y: 0 })
const screenAnchor = ref({ x: 0, y: 0 })
const anchorElement = ref<HTMLElement | null>(null)
let openedAt = 0

const { menuOptions, bump } = useMoreOptionsMenu()
const canvasStore = useCanvasStore()

const lgCanvas = canvasStore.getCanvas()
const { left: canvasLeft, top: canvasTop } = useElementBounding(lgCanvas.canvas)

function syncScreenAnchor() {
  if (anchorElement.value) {
    const rect = anchorElement.value.getBoundingClientRect()
    screenAnchor.value = { x: rect.left, y: rect.bottom }
    return
  }
  const { scale, offset } = lgCanvas.ds
  screenAnchor.value = {
    x: (worldPosition.value.x + offset[0]) * scale + canvasLeft.value,
    y: (worldPosition.value.y + offset[1]) * scale + canvasTop.value
  }
}

const { resume: startSync, pause: stopSync } = useRafFn(syncScreenAnchor, {
  immediate: false
})

watch(isOpen, (open) => {
  if (open) startSync()
  else stopSync()
})

const anchorStyle = computed(() => ({
  left: `${screenAnchor.value.x}px`,
  top: `${screenAnchor.value.y}px`
}))

function show(event: MouseEvent) {
  bump()
  if (event.type === 'contextmenu') {
    anchorElement.value = null
    const screenX = event.clientX - canvasLeft.value
    const screenY = event.clientY - canvasTop.value
    const { scale, offset } = lgCanvas.ds
    worldPosition.value = {
      x: screenX / scale - offset[0],
      y: screenY / scale - offset[1]
    }
  } else {
    anchorElement.value =
      event.currentTarget instanceof HTMLElement ? event.currentTarget : null
  }
  syncScreenAnchor()
  openedAt = performance.now()
  isOpen.value = true
}

// A right-click on a widget input focuses it immediately after the menu
// opens, which reka treats as a focus-out and auto-dismisses. Suppress
// only that opening-time focus steal; later focus-outs (clicking the
// canvas, another node) still dismiss normally.
function onFocusOutside(event: Event) {
  if (performance.now() - openedAt < OPEN_FOCUS_GRACE_MS) {
    event.preventDefault()
  }
}

function hide() {
  isOpen.value = false
}

function toggle(event: Event) {
  if (isOpen.value) hide()
  else show(event as MouseEvent)
}

defineExpose({ toggle, hide, isOpen, show })

function runAction(option: MenuOption) {
  if (option.disabled) return
  option.action?.()
  hide()
}

function handleSubmenuSelect(sub: SubMenuOption) {
  if (sub.disabled) return
  sub.action()
  hide()
}

function optionKey(option: MenuOption, idx: number): string {
  return option.type === 'divider'
    ? `sep-${idx}`
    : `${option.label ?? ''}-${idx}`
}

onMounted(() => {
  registerNodeOptionsInstance({ toggle, show, hide, isOpen })
})

onUnmounted(() => {
  registerNodeOptionsInstance(null)
})
</script>
