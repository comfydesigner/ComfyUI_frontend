<script setup lang="ts">
import DropdownMenuItem from '@/components/ui/dropdown-menu/DropdownMenuItem.vue'
import DropdownMenuSeparator from '@/components/ui/dropdown-menu/DropdownMenuSeparator.vue'
import DropdownMenuShortcut from '@/components/ui/dropdown-menu/DropdownMenuShortcut.vue'
import DropdownMenuSub from '@/components/ui/dropdown-menu/DropdownMenuSub.vue'
import DropdownMenuSubContent from '@/components/ui/dropdown-menu/DropdownMenuSubContent.vue'
import DropdownMenuSubTrigger from '@/components/ui/dropdown-menu/DropdownMenuSubTrigger.vue'
import type {
  MenuOption,
  SubMenuOption
} from '@/composables/graph/useMoreOptionsMenu'
import { useNodeCustomization } from '@/composables/graph/useNodeCustomization'

defineOptions({ name: 'NodeContextMenuItem' })

const { option } = defineProps<{ option: MenuOption }>()

const emit = defineEmits<{
  select: [option: MenuOption]
  'submenu-select': [sub: SubMenuOption]
}>()

const { getCurrentShape } = useNodeCustomization()

function isShapeSelected(sub: SubMenuOption): boolean {
  if (sub.color) return false
  const currentShape = getCurrentShape()
  if (!currentShape) return false
  return currentShape.localizedName === sub.label
}
</script>

<template>
  <DropdownMenuSeparator v-if="option.type === 'divider'" />

  <DropdownMenuSub v-else-if="option.subOptions">
    <DropdownMenuSubTrigger>
      <template v-if="option.icon" #icon><i :class="option.icon" /></template>
      {{ option.label }}
    </DropdownMenuSubTrigger>
    <DropdownMenuSubContent>
      <NodeContextMenuItem
        v-for="(child, idx) in option.subOptions"
        :key="`${child.label ?? ''}-${idx}`"
        :option="child"
        @select="(o) => emit('select', o)"
        @submenu-select="(s) => emit('submenu-select', s)"
      />
    </DropdownMenuSubContent>
  </DropdownMenuSub>

  <DropdownMenuSub v-else-if="option.isColorPicker">
    <DropdownMenuSubTrigger>
      <template v-if="option.icon" #icon><i :class="option.icon" /></template>
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
          @click="emit('submenu-select', sub)"
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
      <template v-if="option.icon" #icon><i :class="option.icon" /></template>
      {{ option.label }}
    </DropdownMenuSubTrigger>
    <DropdownMenuSubContent>
      <DropdownMenuItem
        v-for="sub in option.submenu ?? []"
        :key="sub.label"
        :disabled="sub.disabled"
        checkable
        :checked="isShapeSelected(sub)"
        @select="emit('submenu-select', sub)"
      >
        {{ sub.label }}
      </DropdownMenuItem>
    </DropdownMenuSubContent>
  </DropdownMenuSub>

  <DropdownMenuSub v-else-if="option.hasSubmenu && option.submenu">
    <DropdownMenuSubTrigger>
      <template v-if="option.icon" #icon><i :class="option.icon" /></template>
      {{ option.label }}
    </DropdownMenuSubTrigger>
    <DropdownMenuSubContent>
      <DropdownMenuItem
        v-for="sub in option.submenu"
        :key="sub.label"
        :disabled="sub.disabled"
        @select="emit('submenu-select', sub)"
      >
        <template v-if="sub.icon" #icon><i :class="sub.icon" /></template>
        {{ sub.label }}
      </DropdownMenuItem>
    </DropdownMenuSubContent>
  </DropdownMenuSub>

  <DropdownMenuItem
    v-else
    :disabled="option.disabled"
    @select="emit('select', option)"
  >
    <template v-if="option.icon" #icon><i :class="option.icon" /></template>
    {{ option.label }}
    <DropdownMenuShortcut v-if="option.shortcut">
      {{ option.shortcut }}
    </DropdownMenuShortcut>
  </DropdownMenuItem>
</template>
