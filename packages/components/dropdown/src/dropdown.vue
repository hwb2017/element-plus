<template>
  <div class="el-dropdown">
    <el-popper
      ref="triggerVnode"
      v-model:visible="visible"
      :placement="placement"
      :fallback-placements="['bottom', 'top', 'right', 'left']"
      :effect="effect"
      pure
      :manual-mode="true"
      :trigger="[trigger]"
      popper-class="el-dropdown__popper"
      append-to-body
      transition="el-zoom-in-top"
      :stop-popper-mouse-event="false"
      :gpu-acceleration="false"
    >
      <template #default>
        <el-scrollbar
          ref="scrollbar"
          tag="ul"
          :wrap-style="wrapStyle"
          view-class="el-dropdown__list"
        >
          <slot name="dropdown"></slot>
        </el-scrollbar>
      </template>
      <template #trigger>
        <div :class="[dropdownSize ? 'el-dropdown--' + dropdownSize : '']">
          <slot v-if="!splitButton" name="default"></slot>
          <template v-else>
            <el-button-group>
              <el-button
                :size="dropdownSize"
                :type="type"
                @click="handlerMainButtonClick"
              >
                <slot name="default"></slot>
              </el-button>
              <el-button
                :size="dropdownSize"
                :type="type"
                class="el-dropdown__caret-button"
              >
                <el-icon class="el-dropdown__icon"><arrow-down /></el-icon>
              </el-button>
            </el-button-group>
          </template>
        </div>
      </template>
    </el-popper>
  </div>
</template>
<script lang="ts">
import {
  defineComponent,
  provide,
  getCurrentInstance,
  ref,
  computed,
  watch,
  onMounted,
} from 'vue'
import ElButton from '@element-plus/components/button'
import ElPopper, { Effect } from '@element-plus/components/popper'
import ElScrollbar from '@element-plus/components/scrollbar'
import ElIcon from '@element-plus/components/icon'
import { on, addClass, removeClass } from '@element-plus/utils/dom'
import { addUnit } from '@element-plus/utils/util'
import { ArrowDown } from '@element-plus/icons-vue'
import { useSize } from '@element-plus/hooks'

import type { Placement } from '@element-plus/components/popper'
import type { PropType, ComponentPublicInstance, CSSProperties } from 'vue'
import type { TriggerType } from '@element-plus/hooks/use-popper/use-target-events'
import type { ButtonType } from '@element-plus/components/button/src/types'

type Nullable<T> = null | T
const { ButtonGroup: ElButtonGroup } = ElButton

export default defineComponent({
  name: 'ElDropdown',
  components: {
    ElButton,
    ElButtonGroup,
    ElScrollbar,
    ElPopper,
    ElIcon,
    ArrowDown,
  },
  props: {
    trigger: {
      type: String as PropType<TriggerType | 'contextmenu'>,
      default: 'hover',
    },
    type: String as PropType<ButtonType>,
    size: {
      type: String,
      default: '',
    },
    splitButton: Boolean,
    hideOnClick: {
      type: Boolean,
      default: true,
    },
    placement: {
      type: String as PropType<Placement>,
      default: 'bottom',
    },
    showTimeout: {
      type: Number,
      default: 150,
    },
    hideTimeout: {
      type: Number,
      default: 150,
    },
    tabindex: {
      type: [Number, String],
      default: 0,
    },
    effect: {
      type: String as PropType<Effect>,
      default: Effect.LIGHT,
    },
    maxHeight: {
      type: [Number, String],
      default: '',
    },
  },
  emits: ['visible-change', 'click', 'command'],
  setup(props, { emit }) {
    const _instance = getCurrentInstance()

    const timeout = ref<Nullable<number>>(null)

    const visible = ref(false)
    const scrollbar = ref(null)
    const wrapStyle = computed<CSSProperties>(() => ({
      maxHeight: addUnit(props.maxHeight),
    }))

    watch(
      () => visible.value,
      (val) => {
        if (val) triggerElmFocus()
        if (!val) triggerElmBlur()
        emit('visible-change', val)
      }
    )

    const focusing = ref(false)
    watch(
      () => focusing.value,
      (val) => {
        const selfDefine = triggerElm.value
        if (selfDefine) {
          if (val) {
            addClass(selfDefine, 'focusing')
          } else {
            removeClass(selfDefine, 'focusing')
          }
        }
      }
    )

    const triggerVnode = ref<Nullable<ComponentPublicInstance>>(null)
    const triggerElm = computed<Nullable<HTMLButtonElement>>(() => {
      const _: any = (triggerVnode.value?.$refs.triggerRef as HTMLElement)
        ?.children[0]
      return !props.splitButton ? _ : _?.children?.[1]
    })

    function handleClick() {
      if (triggerElm.value?.disabled) return
      if (visible.value) {
        hide()
      } else {
        show()
      }
    }

    function show() {
      if (triggerElm.value?.disabled) return
      timeout.value && clearTimeout(timeout.value)
      timeout.value = window.setTimeout(
        () => {
          visible.value = true
        },
        ['click', 'contextmenu'].includes(props.trigger) ? 0 : props.showTimeout
      )
    }

    function hide() {
      if (triggerElm.value?.disabled) return
      removeTabindex()
      if (props.tabindex >= 0) {
        resetTabindex(triggerElm.value)
      }
      clearTimeout(timeout.value)
      timeout.value = window.setTimeout(
        () => {
          visible.value = false
        },
        ['click', 'contextmenu'].includes(props.trigger) ? 0 : props.hideTimeout
      )
    }

    function removeTabindex() {
      triggerElm.value?.setAttribute('tabindex', '-1')
    }

    function resetTabindex(ele) {
      removeTabindex()
      ele?.setAttribute('tabindex', '0')
    }

    function triggerElmFocus() {
      triggerElm.value?.focus?.()
    }

    function triggerElmBlur() {
      triggerElm.value?.blur?.()
    }

    const dropdownSize = useSize()

    function commandHandler(...args) {
      emit('command', ...args)
    }

    provide('elDropdown', {
      instance: _instance,
      dropdownSize,
      visible,
      handleClick,
      commandHandler,
      show,
      hide,
      trigger: computed(() => props.trigger),
      hideOnClick: computed(() => props.hideOnClick),
      triggerElm,
    })

    onMounted(() => {
      if (!props.splitButton) {
        on(triggerElm.value, 'focus', () => {
          focusing.value = true
        })
        on(triggerElm.value, 'blur', () => {
          focusing.value = false
        })
        on(triggerElm.value, 'click', () => {
          focusing.value = false
        })
      }
      if (props.trigger === 'hover') {
        on(triggerElm.value, 'mouseenter', show)
        on(triggerElm.value, 'mouseleave', hide)
      } else if (props.trigger === 'click') {
        on(triggerElm.value, 'click', handleClick)
      } else if (props.trigger === 'contextmenu') {
        on(triggerElm.value, 'contextmenu', (e) => {
          e.preventDefault()
          handleClick()
        })
      }

      Object.assign(_instance, {
        handleClick,
        hide,
        resetTabindex,
      })
    })

    const handlerMainButtonClick = (event) => {
      emit('click', event)
      hide()
    }

    return {
      visible,
      scrollbar,
      wrapStyle,
      dropdownSize,
      handlerMainButtonClick,
      triggerVnode,
    }
  },
})
</script>
