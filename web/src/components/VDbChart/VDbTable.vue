<template>
  <svg
    ref="root"
    :id="`table-${id}`"
    :class="{
      'db-table':true,
      'db-table__highlight': highlight,
      'db-table__dragging': dragging
    }"
    :x="state.x"
    :y="state.y"
    :width="state.width"
    :height="state.height"
    @mouseenter.passive="onMouseEnter"
    @mouseleave.passive="onMouseLeave"
  >
    <rect class="db-table__background"
          :width="state.width"
          :height="state.height"
    />
    <g class="db-table-header"
       @mouseenter.passive="showTooltip"
       @mouseleave.passive="hideTooltip"
    >
      <rect
        height="35"
        :width="state.width"
        :fill="headerColor"
        @mousedown.stop="startDrag"
        @click.passive="onHeaderClick"
      />
      <text class="db-table-header__name"
            y="16"
      >
        {{ name }}
      </text>
      <title>{{ name }}</title>
    </g>
    <g class="db-table-fields">
      <v-db-field v-for="field of fields"
                  v-bind="field"
                  :key="field.id"
                  :width="state.width"
                  @click.passive="onFieldClick($event, field)"
      />
    </g>
  </svg>
</template>

<script setup>
  import { computed, onMounted, ref, watch } from 'vue'
  import VDbField from './VDbField'
  import VDbTableTooltip from './VDbTableTooltip'
  import { useChartStore } from '../../store/chart'
  import { snap } from '../../utils/MathUtil'
  import { useEditorStore } from '../../store/editor'

  const props = defineProps({
    id: Number,
    selection: String,
    token: Object,
    group: Object,
    name: String,
    alias: String,
    note: String,
    indexes: Array,
    schema: Object,
    headerColor: {
      type: String,
      default: () => ('')
    },
    dbState: Object,
    fields: {
      type: Array,
      default: () => ([])
    },
    containerRef: Object
  })

  const editor = useEditorStore()
  const store = useChartStore()

  const state = computed(() => store.getTable(props.id))

  const root = ref(null)

  const updateWidth = () => {
    if(!root.value) return;
    const fieldEls = [...root.value.querySelectorAll('.db-field')];
    const maxFieldWidth = fieldEls.map(f => [...f.querySelectorAll('text')].map(ft => ft.getComputedTextLength())
      .reduce((prev,curr) => prev + curr, 3*16))
      .reduce((prev,curr) => Math.max(prev, curr), 0);

    state.value.width = snap(Math.max(200, maxFieldWidth), gridSnap);
  }

  const updateHeight = () => {
    state.value.height = 35 + (30 * props.fields.length);
  }

  watch(() => props.fields, value => {
    updateHeight();
    updateWidth();
  });

  onMounted(() => {
    updateHeight();
    updateWidth();
  })

  const emit = defineEmits([
    'update:position',
    'click:header',
    'click:field'
  ])


  const tooltipSize = computed(() => ({
    width: 200,
    height: 35 + (30 * props.fields.length)
  }))

  const highlight = ref(false)
  const tooltip = ref(false)
  const dragging = ref(false)
  const dragOffsetX = ref(null)
  const dragOffsetY = ref(null)
  const dragOffset = ref(null)
  const gridSize = store.subGridSize;
  const gridSnap = store.grid.snap;

  const onMouseEnter = (e) => {
    highlight.value = true
    console.log('[table] mouseenter fired on', props.name)
  }
  const onMouseLeave = (e) => {
    highlight.value = false
    dragging.value = false
  }

  const toSVGPoint = (clientX, clientY) => {
    const pt = props.containerRef.createSVGPoint()
    pt.x = clientX
    pt.y = clientY
    return pt.matrixTransform(root.value.parentElement.getScreenCTM().inverse())
  }

  const drag = (event) => {
    const p = toSVGPoint(event.clientX, event.clientY)
    state.value.x = snap(p.x - dragOffsetX.value, gridSnap)
    state.value.y = snap(p.y - dragOffsetY.value, gridSnap)
    emit('update:position', state.value)
  }
  const drop = (e) => {
    dragging.value = false
    highlight.value = false

    dragOffsetX.value = null
    dragOffsetY.value = null
    props.containerRef.removeEventListener('mousemove', drag, { passive: true })
    props.containerRef.removeEventListener('mouseup', drop, { passive: true })
    props.containerRef.removeEventListener('mouseleave', onMouseLeave, { passive: true })
  }
  const startDrag = (event) => {
    dragging.value = true
    console.log('[drag] startDrag fired', event.clientX, event.clientY)
    console.log('[drag] containerRef:', props.containerRef)
    console.log('[drag] root.value.parentElement:', root.value?.parentElement)
    console.log('[drag] getScreenCTM:', root.value?.parentElement?.getScreenCTM())

    const p = toSVGPoint(event.clientX, event.clientY)
    console.log('[drag] svgPoint:', p.x, p.y, 'state:', state.value.x, state.value.y)
    dragOffsetX.value = p.x - state.value.x
    dragOffsetY.value = p.y - state.value.y

    props.containerRef.addEventListener('mousemove', drag, { passive: true })
    props.containerRef.addEventListener('mouseup', drop, { passive: true })
    props.containerRef.addEventListener('mouseleave', onMouseLeave, { passive: true })
  }

  const showTooltip = () => {
    const tooltipPosition = {
      x: state.value.x + state.value.width,
      y: state.value.y,
    }
    store.showTooltip(tooltipPosition, VDbTableTooltip, {
      table: props
    })
  }

  const hideTooltip = () => {
    store.hideTooltip();
  }
  function onHeaderClick (e) {
    emit('click:header', e, editor.findTable(props.id));
  }
  function onFieldClick (e, field) {
    emit('click:field', e, field);
  }
</script>

<style scoped>

</style>
