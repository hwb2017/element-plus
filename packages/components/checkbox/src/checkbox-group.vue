<template>
  <div class="el-checkbox-group" role="group" aria-label="checkbox-group">
    <slot></slot>
  </div>
</template>

<script lang="ts">
import {
  defineComponent,
  computed,
  watch,
  provide,
  nextTick,
  toRefs,
} from 'vue'
import { UPDATE_MODEL_EVENT } from '@element-plus/utils/constants'
import { isValidComponentSize } from '@element-plus/utils/validators'
import { useSize } from '@element-plus/hooks'
import { useCheckboxGroup } from './useCheckbox'

import type { PropType } from 'vue'
import type { ComponentSize } from '@element-plus/utils/types'

export default defineComponent({
  name: 'ElCheckboxGroup',

  props: {
    modelValue: {
      type: [Object, Boolean, Array],
      default: () => undefined,
    },
    disabled: Boolean,
    min: {
      type: Number,
      default: undefined,
    },
    max: {
      type: Number,
      default: undefined,
    },
    size: {
      type: String as PropType<ComponentSize>,
      validator: isValidComponentSize,
    },
    fill: {
      type: String,
      default: undefined,
    },
    textColor: {
      type: String,
      default: undefined,
    },
  },

  emits: [UPDATE_MODEL_EVENT, 'change'],

  setup(props, ctx) {
    const { elFormItem } = useCheckboxGroup()
    const checkboxGroupSize = useSize()

    const changeEvent = (value) => {
      ctx.emit(UPDATE_MODEL_EVENT, value)
      nextTick(() => {
        ctx.emit('change', value)
      })
    }

    const modelValue = computed({
      get() {
        return props.modelValue
      },
      set(val) {
        changeEvent(val)
      },
    })

    provide('CheckboxGroup', {
      name: 'ElCheckboxGroup',
      modelValue,
      ...toRefs(props),
      checkboxGroupSize,
      changeEvent,
    })

    watch(
      () => props.modelValue,
      () => {
        elFormItem.validate?.('change')
      }
    )
  },
})
</script>
