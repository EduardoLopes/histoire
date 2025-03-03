<script lang="ts">
export default {
  name: 'HstButtonGroup',
}
</script>

<script setup lang="ts">
import { computed, ComputedRef } from 'vue'
import HstWrapper from '../HstWrapper.vue'
import { HstControlOption } from '../../types'
import HstButton from './HstButton.vue'

const props = defineProps<{
  title?: string
  modelValue: string
  options: string[] | HstControlOption[]
}>()

const formattedOptions: ComputedRef<Record<string, string>> = computed(() => {
  if (Array.isArray(props.options)) {
    return Object.fromEntries(props.options.map((value: string | HstControlOption) => {
      if (typeof value === 'string') {
        return [value, value]
      } else {
        return [value.value, value.label]
      }
    }))
  }
  return props.options
})

const emit = defineEmits<{
  (e: 'update:modelValue', value: string): void
}>()

function selectOption (value: string) {
  emit('update:modelValue', value)
}
</script>

<template>
  <HstWrapper
    tag="div"
    role="group"
    :title="title"
    class="histoire-button-group htw-flex-nowrap htw-items-center"
  >
    <div class="htw-flex htw-gap-px htw-border htw-border-solid htw-border-black/25 dark:htw-border-white/25 htw-rounded-sm htw-p-px">
      <HstButton
        v-for="( label, value ) in formattedOptions"
        :key="value"
        class="htw-px-1 htw-h-[22px] htw-flex-1 !htw-rounded-[3px]"
        :color="value === modelValue ? 'primary' : 'flat'"
        :rounded="false"
        @click="selectOption(value)"
      >
        {{ label }}
      </HstButton>
    </div>
    <template #actions>
      <slot name="actions" />
    </template>
  </HstWrapper>
</template>
