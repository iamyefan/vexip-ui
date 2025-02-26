<template>
  <li
    ref="wrapper"
    :class="className"
    role="menuitem"
    tabindex="0"
    :aria-disabled="props.disabled ? 'true' : undefined"
    @keydown.enter.stop="handleSelect"
  >
    <Tooltip
      placement="right"
      :theme="tooltipTheme"
      :transfer="true"
      :disabled="tooltipDisabled"
    >
      <template #trigger>
        <div
          ref="reference"
          :class="{
            [nh.be('label')]: true,
            [nh.bem('label', 'in-popper')]: isUsePopper
          }"
          :style="labelStyle"
          @click="handleSelect"
          @mouseenter="handleMouseEnter"
          @mouseleave="handleMouseLeave"
        >
          <div v-if="props.icon" :class="nh.be('icon')">
            <slot name="icon">
              <Icon v-bind="props.iconProps" :icon="props.icon"></Icon>
            </slot>
          </div>
          <span
            :class="{
              [nh.be('title')]: true,
              [nh.bem('title', 'in-group')]: !isHorizontal && isGroup
            }"
          >
            <slot>
              {{ props.label }}
            </slot>
          </span>
          <Icon
            v-if="isGroup"
            :class="{
              [nh.be('arrow')]: true,
              [nh.bem('arrow', 'visible')]: groupExpanded
            }"
          >
            <ChevronDown></ChevronDown>
          </Icon>
        </div>
      </template>
      <span :class="nh.be('tooltip-title')">
        <slot>
          {{ props.label }}
        </slot>
      </span>
    </Tooltip>
    <span v-if="!isUsePopper">
      <CollapseTransition>
        <ul v-show="showGroup" :class="[nh.be('list'), nh.bem('list', 'theme')]">
          <slot name="group">
            <Renderer :renderer="renderChildren"></Renderer>
          </slot>
        </ul>
      </CollapseTransition>
    </span>
    <Portal v-else-if="isGroup" :to="transferTo">
      <transition :name="transition">
        <div
          v-show="showGroup"
          ref="popper"
          :class="[nh.be('popper'), nh.bs('vars'), isHorizontal ? nh.bem('popper', 'drop') : '']"
          @mouseenter="handleMouseEnter"
          @mouseleave="handleMouseLeave"
        >
          <ul :class="[nh.be('list'), nh.bem('list', 'theme')]">
            <slot name="group">
              <Renderer :renderer="renderChildren"></Renderer>
            </slot>
          </ul>
        </div>
      </transition>
    </Portal>
  </li>
</template>

<script lang="tsx">
import {
  defineComponent,
  ref,
  reactive,
  computed,
  inject,
  provide,
  toRef,
  watch,
  onMounted,
  onBeforeUnmount,
  nextTick
} from 'vue'
import { CollapseTransition } from '@/components/collapse-transition'
import { Icon } from '@/components/icon'
import { MenuGroup } from '@/components/menu-group'
import { Portal } from '@/components/portal'
import { Tooltip } from '@/components/tooltip'
import { Renderer } from '@/components/renderer'
import {
  useNameHelper,
  useProps,
  booleanProp,
  booleanStringProp,
  eventProp,
  emitEvent
} from '@vexip-ui/config'
import { usePopper, useSetTimeout, useClickOutside } from '@vexip-ui/mixins'
import { ChevronDown } from '@vexip-ui/icons'
import { baseIndentWidth, MENU_STATE, MENU_ITEM_STATE, MENU_GROUP_STATE } from './symbol'

import type { PropType } from 'vue'
import type { RouteLocationRaw } from 'vue-router'
import type { Placement } from '@vexip-ui/mixins'
import type { IconMinorProps } from '@/components/icon'
import type { MenuOptions } from './symbol'

const MenuItem = defineComponent({
  name: 'MenuItem',
  components: {
    CollapseTransition,
    Icon,
    Tooltip,
    Portal,
    Renderer,
    ChevronDown
  },
  props: {
    label: String,
    icon: Object,
    iconProps: Object as PropType<IconMinorProps>,
    disabled: booleanProp,
    transfer: booleanStringProp,
    trigger: String as PropType<'hover' | 'click'>,
    transitionName: String,
    meta: Object,
    children: Array as PropType<MenuOptions[]>,
    route: [String, Object] as PropType<RouteLocationRaw>,
    onSelect: eventProp()
  },
  emits: [],
  setup(_props, { slots }) {
    const props = useProps('menuItem', _props, {
      label: {
        default: null,
        static: true
      },
      icon: null,
      iconProps: null,
      disabled: {
        default: false,
        static: true
      },
      transfer: null,
      trigger: null,
      transitionName: null,
      meta: null,
      children: {
        default: () => [],
        static: true
      },
      route: null
    })

    const menuState = inject(MENU_STATE, null)
    const parentItemState = inject(MENU_ITEM_STATE, null)
    const groupState = inject(MENU_GROUP_STATE, null)

    const nh = useNameHelper('menu')
    const baseClass = nh.be('item')
    const placement = ref('right-start' as Placement)
    const groupExpanded = ref(false)
    const selected = ref(false)
    const sonSelected = ref(false)

    const indent = computed(() => (parentItemState?.indent ?? 0) + 1)
    const propTransfer = computed(() => props.transfer ?? menuState?.transfer ?? false)
    const inTransfer = computed(() => (parentItemState ? parentItemState.transfer : false))
    const transfer = computed(() => !inTransfer.value && propTransfer.value)

    const wrapper = useClickOutside(handleClickOutside)
    const { reference, popper, transferTo, updatePopper } = usePopper({
      placement,
      transfer,
      wrapper
    })

    const className = computed(() => {
      return {
        [baseClass]: true,
        [`${baseClass}--disabled`]: props.disabled,
        [`${baseClass}--group-visible`]: groupExpanded.value,
        [`${baseClass}--selected`]: selected.value,
        [`${baseClass}--no-icon`]: !props.icon,
        [`${baseClass}--son-selected`]: sonSelected.value
      }
    })
    const labelStyle = computed(() => {
      if (menuState?.horizontal || parentItemState?.isUsePopper) {
        return {}
      }

      return {
        paddingLeft:
          parentItemState && parentItemState.isUsePopper
            ? undefined
            : `${
                indent.value * baseIndentWidth + (groupState?.indent ?? 0) * 0.25 * baseIndentWidth
              }px`
      }
    })
    const isGroup = computed(() => !!(slots.group || props.children?.length))
    const showGroup = computed(() => isGroup.value && groupExpanded.value)
    const isUsePopper = computed(() => {
      return (
        (menuState && (menuState.horizontal || menuState.groupType === 'dropdown')) ||
        (isGroup.value && menuState?.isReduced && !parentItemState) ||
        !!parentItemState?.isUsePopper
      )
    })
    const tooltipDisabled = computed(() => {
      return (
        isGroup.value || !!(parentItemState?.isUsePopper || (menuState && !menuState.isReduced))
      )
    })
    const theme = computed(() => (menuState ? menuState.theme : 'light'))
    const tooltipTheme = computed(() => (menuState ? menuState.tooltipTheme : 'dark'))
    const isHorizontal = computed(() => menuState?.horizontal && !parentItemState)
    const transition = computed(() => {
      return props.transitionName ?? isHorizontal.value ? nh.ns('drop') : nh.ns('zoom')
    })
    const dropTrigger = computed(() => props.trigger || menuState?.trigger || 'hover')

    const itemState = reactive({
      el: wrapper,
      label: toRef(props, 'label'),
      indent,
      groupExpanded,
      isUsePopper,
      parentState: parentItemState,
      transfer: computed(() => inTransfer.value || propTransfer.value),
      updateSonSelected,
      toggleGroupExpanded,
      handleMouseEnter,
      handleMouseLeave
    })

    provide(MENU_ITEM_STATE, itemState)

    watch(showGroup, value => {
      if (value && isUsePopper.value) {
        updatePopper()
      }
    })
    watch(selected, value => {
      if (value) {
        emitEvent(props.onSelect)
        nextTick(() => {
          parentItemState?.updateSonSelected(value)
        })
      } else {
        parentItemState?.updateSonSelected(value)
      }
    })
    watch(groupExpanded, expanded => {
      if (typeof menuState?.handleExpand === 'function') {
        menuState.handleExpand(props.label, expanded, props.meta || {})
      }
    })
    watch(
      isHorizontal,
      value => {
        placement.value = value ? 'bottom' : 'right-start'
      },
      { immediate: true }
    )

    if (menuState) {
      watch(
        () => menuState.currentActive,
        value => {
          selected.value = props.label === value
        },
        { immediate: true }
      )
    }

    onMounted(() => {
      if (typeof menuState?.increaseItem === 'function') {
        menuState.increaseItem(itemState)
      }
    })

    onBeforeUnmount(() => {
      if (typeof menuState?.decreaseItem === 'function') {
        menuState.decreaseItem(itemState)
      }
    })

    function updateSonSelected(selected: boolean) {
      sonSelected.value = selected
      parentItemState?.updateSonSelected(selected)
    }

    const { timer } = useSetTimeout()

    function handleSelect() {
      window.clearTimeout(timer.hover)

      if (props.disabled) return

      if (isGroup.value) {
        if (isUsePopper.value && dropTrigger.value !== 'click') return

        groupExpanded.value = !groupExpanded.value
      } else {
        if (isUsePopper.value) {
          toggleGroupExpanded(false, true)
        }

        if (menuState) {
          menuState.handleSelect(props.label, props.meta || {}, props.route)
        }

        selected.value = true
      }
    }

    function toggleGroupExpanded(expanded: boolean, upwrad = false) {
      window.clearTimeout(timer.hover)

      groupExpanded.value = expanded

      if (upwrad && typeof parentItemState?.toggleGroupExpanded === 'function') {
        parentItemState.toggleGroupExpanded(expanded, upwrad)
      }
    }

    function handleMouseEnter() {
      window.clearTimeout(timer.hover)

      if (props.disabled || !isUsePopper.value || dropTrigger.value !== 'hover') return

      if (typeof parentItemState?.handleMouseEnter === 'function') {
        parentItemState.handleMouseEnter()
      }

      if (!isGroup.value) return

      timer.hover = window.setTimeout(() => {
        groupExpanded.value = true
      }, 250)
    }

    function handleMouseLeave() {
      window.clearTimeout(timer.hover)

      if (props.disabled || !isUsePopper.value || dropTrigger.value !== 'hover') return

      if (typeof parentItemState?.handleMouseLeave === 'function') {
        parentItemState.handleMouseLeave()
      }

      if (!isGroup.value) return

      timer.hover = window.setTimeout(() => {
        groupExpanded.value = false
      }, 250)
    }

    function handleClickOutside() {
      if (dropTrigger.value === 'click') {
        nextTick(() => {
          groupExpanded.value = false
        })
      }
    }

    function renderChildren() {
      if (!props.children?.length) {
        return null
      }

      const renderItem = (item: MenuOptions) => (
        <MenuItem
          label={item.label}
          icon={item.icon}
          icon-props={item.iconProps}
          disabled={item.disabled}
          children={item.children}
          route={item.route}
        >
          {item.name || item.label}
        </MenuItem>
      )

      return props.children.map(child => {
        if (child.group) {
          return (
            <MenuGroup label={child.name || child.label}>
              {child.children?.map(renderItem)}
            </MenuGroup>
          )
        }

        return renderItem(child)
      })
    }

    return {
      props,
      nh,
      groupExpanded,
      transferTo,

      className,
      labelStyle,
      isGroup,
      showGroup,
      isUsePopper,
      tooltipDisabled,
      theme,
      tooltipTheme,
      isHorizontal,
      transition,
      dropTrigger,

      wrapper,
      reference,
      popper,

      handleSelect,
      handleMouseEnter,
      handleMouseLeave,
      renderChildren
    }
  }
})

// eslint-disable-next-line vue/require-direct-export
export default MenuItem
</script>
