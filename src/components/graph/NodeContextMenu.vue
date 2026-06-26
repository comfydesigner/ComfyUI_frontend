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
    >
      <template
        v-for="(option, idx) in menuOptions"
        :key="optionKey(option, idx)"
      >
        <DropdownMenuSeparator v-if="option.type === 'divider'" />

        <DropdownMenuSub v-else-if="option.isColorPicker">
          <DropdownMenuSubTrigger>
            <template v-if="option.icon" #icon
              ><i :class="option.icon"
            /></template>
            {{ option.label }}
          </DropdownMenuSubTrigger>
          <DropdownMenuSubContent class="p-2">
            <div class="flex flex-col gap-1">
              <button
                v-for="sub in option.submenu ?? []"
                :key="sub.label"
                type="button"
                :title="sub.label"
                :disabled="sub.disabled"
                class="flex size-7 items-center justify-center rounded-sm hover:bg-secondary-background-hover disabled:cursor-not-allowed disabled:opacity-50"
                @click="handleSubmenuSelect(sub)"
              >
                <div
                  class="size-5 rounded-full border border-border-default"
                  :style="{ backgroundColor: sub.color }"
                />
              </button>
            </div>
          </DropdownMenuSubContent>
        </DropdownMenuSub>

        <DropdownMenuSub v-else-if="option.isShapePicker">
          <DropdownMenuSubTrigger>
            <template v-if="option.icon" #icon
              ><i :class="option.icon"
            /></template>
            {{ option.label }}
          </DropdownMenuSubTrigger>
          <DropdownMenuSubContent>
            <DropdownMenuItem
              v-for="sub in option.submenu ?? []"
              :key="sub.label"
              :disabled="sub.disabled"
              checkable
              :checked="isShapeSelected(sub)"
              @select="handleSubmenuSelect(sub)"
            >
              {{ sub.label }}
            </DropdownMenuItem>
          </DropdownMenuSubContent>
        </DropdownMenuSub>

        <DropdownMenuSub v-else-if="option.hasSubmenu && option.submenu">
          <DropdownMenuSubTrigger>
            <template v-if="option.icon" #icon
              ><i :class="option.icon"
            /></template>
            {{ option.label }}
          </DropdownMenuSubTrigger>
          <DropdownMenuSubContent>
            <DropdownMenuItem
              v-for="sub in option.submenu"
              :key="sub.label"
              :disabled="sub.disabled"
              @select="handleSubmenuSelect(sub)"
            >
              <template v-if="sub.icon" #icon><i :class="sub.icon" /></template>
              {{ sub.label }}
            </DropdownMenuItem>
          </DropdownMenuSubContent>
        </DropdownMenuSub>

        <DropdownMenuItem
          v-else
          :disabled="option.disabled"
          @select="runAction(option)"
        >
          <template v-if="option.icon" #icon
            ><i :class="option.icon"
          /></template>
          {{ option.label }}
          <DropdownMenuShortcut v-if="option.shortcut">
            {{ option.shortcut }}
          </DropdownMenuShortcut>
        </DropdownMenuItem>
      </template>
    </DropdownMenuContent>
  </DropdownMenu>
</template>

<script setup lang="ts">
import { useElementBounding, useRafFn } from '@vueuse/core'
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'

import DropdownMenu from '@/components/ui/dropdown-menu/DropdownMenu.vue'
import DropdownMenuContent from '@/components/ui/dropdown-menu/DropdownMenuContent.vue'
import DropdownMenuItem from '@/components/ui/dropdown-menu/DropdownMenuItem.vue'
import DropdownMenuSeparator from '@/components/ui/dropdown-menu/DropdownMenuSeparator.vue'
import DropdownMenuShortcut from '@/components/ui/dropdown-menu/DropdownMenuShortcut.vue'
import DropdownMenuSub from '@/components/ui/dropdown-menu/DropdownMenuSub.vue'
import DropdownMenuSubContent from '@/components/ui/dropdown-menu/DropdownMenuSubContent.vue'
import DropdownMenuSubTrigger from '@/components/ui/dropdown-menu/DropdownMenuSubTrigger.vue'
import DropdownMenuTrigger from '@/components/ui/dropdown-menu/DropdownMenuTrigger.vue'
import {
  registerNodeOptionsInstance,
  useMoreOptionsMenu
} from '@/composables/graph/useMoreOptionsMenu'
import type {
  MenuOption,
  SubMenuOption
} from '@/composables/graph/useMoreOptionsMenu'
import { useNodeCustomization } from '@/composables/graph/useNodeCustomization'
import { useCanvasStore } from '@/renderer/core/canvas/canvasStore'

const isOpen = ref(false)
const worldPosition = ref({ x: 0, y: 0 })
const screenAnchor = ref({ x: 0, y: 0 })

const { menuOptions, bump } = useMoreOptionsMenu()
const canvasStore = useCanvasStore()
const { getCurrentShape } = useNodeCustomization()

const lgCanvas = canvasStore.getCanvas()
const { left: canvasLeft, top: canvasTop } = useElementBounding(lgCanvas.canvas)

function syncScreenAnchor() {
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
  const screenX = event.clientX - canvasLeft.value
  const screenY = event.clientY - canvasTop.value
  const { scale, offset } = lgCanvas.ds
  worldPosition.value = {
    x: screenX / scale - offset[0],
    y: screenY / scale - offset[1]
  }
  syncScreenAnchor()
  isOpen.value = true
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

function isShapeSelected(sub: SubMenuOption): boolean {
  if (sub.color) return false
  const currentShape = getCurrentShape()
  if (!currentShape) return false
  return currentShape.localizedName === sub.label
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
