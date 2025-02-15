<script>
  import { onMount, setContext, tick, onDestroy } from 'svelte'
  import { writable, derived } from 'svelte/store'
  import moment from 'moment'

  let ganttElement
  let ganttDragElement
  let mainHeaderContainer
  let mainContainer
  let rowContainer
  let scrollables = []
  let mounted = false

  import {
    rowStore,
    taskStore,
    timeRangeStore,
    allTasks,
    allRows,
    allTimeRanges,
    rowTaskCache,
  } from './core/store'
  import { Task, Row, TimeRange, TimeRangeHeader, Milestone } from './entities'
  import { Columns, ColumnHeader } from './column'
  // import { Resizer } from './ui'

  import { GanttUtils, getPositionByDate } from './utils/utils'
  import { getRelativePos, debounce, throttle } from './utils/domUtils'
  import { SelectionManager } from './utils/selectionManager'
  import { GanttApi } from './core/api'
  import { TaskFactory, reflectTask } from './core/task'
  import { RowFactory } from './core/row'
  import { TimeRangeFactory } from './core/timeRange'
  import { DragDropManager } from './core/drag'
  import { findByPosition, findByDate } from './core/column'
  import { onEvent, onDelegatedEvent, offDelegatedEvent } from './core/events'

  export let rows
  export let tasks = []
  export let timeRanges = []
  $: if (mounted) initRows(rows)
  $: if (mounted) initTasks(tasks)
  $: if (mounted) initTimeRanges(timeRanges)

  export let rowPadding = 6
  export let rowHeight = 48
  const _rowHeight = writable(rowHeight)
  const _rowPadding = writable(rowPadding)

  export let from
  export let to
  const _from = writable(from)
  const _to = writable(to)
  $: $_from = from
  $: $_to = to

  export let minWidth = 800
  export let fitWidth = false
  const _minWidth = writable(minWidth)
  const _fitWidth = writable(fitWidth)
  $: {
    $_minWidth = minWidth
    $_fitWidth = fitWidth
  }

  export let classes = []
  export let headers = [
    { unit: 'day', format: 'MMMM Do' },
    { unit: 'hour', format: 'H:mm' },
  ]
  export let zoomLevels = [
    {
      headers: [
        { unit: 'day', format: 'DD.MM.YYYY' },
        { unit: 'hour', format: 'HH' },
      ],
      minWidth: 800,
      fitWidth: true,
    },
    {
      headers: [
        { unit: 'hour', format: 'ddd D/M, H A' },
        { unit: 'minute', format: 'mm', offset: 15 },
      ],
      minWidth: 5000,
      fitWidth: false,
    },
  ]
  export let taskContent = null
  export let tableWidth = 100
  export let resizeHandleWidth = 10
  export let onTaskButtonClick = null

  export let magnetUnit = 'minute'
  export let magnetOffset = 15
  export let columnUnit = 'minute'
  export let columnOffset = 15

  // export until Svelte3 implements Svelte2's setup(component) hook
  export let ganttTableModules = []
  export let ganttBodyModules = []

  export let reflectOnParentRows = true
  export let reflectOnChildRows = false

  export let columnStrokeColor
  export let columnStrokeWidth

  const visibleWidth = writable()
  const visibleHeight = writable()
  const headerHeight = writable()
  const _width = derived(
    [visibleWidth, _minWidth, _fitWidth],
    ([visible, min, stretch]) => {
      return stretch && visible > min ? visible : min
    },
  )

  export const columnService = {
    getColumnByDate(date) {
      const pair = findByDate(columns, date)
      return !pair[0] ? pair[1] : pair[0]
    },
    getColumnByPosition(x) {
      const pair = findByPosition(columns, x)
      return !pair[0] ? pair[1] : pair[0]
    },
    getPositionByDate(date) {
      if (!date) return null
      const column = this.getColumnByDate(date)

      let durationTo = date.diff(column.from, 'milliseconds')
      const position = (durationTo / column.duration) * column.width

      //multiples - skip every nth col, use other duration
      return column.left + position
    },
    getDateByPosition(x) {
      const column = this.getColumnByPosition(x)
      x = x - column.left

      let positionDuration = (column.duration / column.width) * x
      const date = moment(column.from).add(positionDuration, 'milliseconds')

      return date
    },
    /**
     *
     * @param {Moment} date - Date
     * @returns {Moment} rounded date passed as parameter
     */
    roundTo(date) {
      let value = date.get(magnetUnit)
      value = Math.round(value / magnetOffset)
      date.set(magnetUnit, value * magnetOffset)

      //round all smaller units to 0
      const units = [
        'millisecond',
        'second',
        'minute',
        'hour',
        'date',
        'month',
        'year',
      ]
      const indexOf = units.indexOf(magnetUnit)
      for (let i = 0; i < indexOf; i++) {
        date.set(units[i], 0)
      }
      return date
    },
  }

  const columnWidth = writable(
    getPositionByDate(
      $_from.clone().add(columnOffset, columnUnit),
      $_from,
      $_to,
      $_width,
    ) | 0,
  )
  $: $columnWidth =
    getPositionByDate(
      $_from.clone().add(columnOffset, columnUnit),
      $_from,
      $_to,
      $_width,
    ) | 0
  let columnCount = Math.ceil($_width / $columnWidth)
  $: columnCount = Math.ceil($_width / $columnWidth)
  let columns = getColumns(
    $_from,
    columnCount,
    columnOffset,
    columnUnit,
    $columnWidth,
  )
  $: columns = getColumns(
    $_from,
    columnCount,
    columnOffset,
    columnUnit,
    $columnWidth,
  )

  function getColumns(from, count, offset, unit, width) {
    let columns = []
    let columnFrom = from.clone()
    let left = 0
    for (let i = 0; i < count; i++) {
      const from = columnFrom.clone()
      const to = columnFrom.add(offset, unit)
      const duration = to.diff(from, 'milliseconds')

      columns.push({
        width: width,
        from,
        left,
        duration,
      })
      left += width
      columnFrom = to
    }

    return columns
  }

  const dimensionsChanged = derived([columnWidth, _from, _to], () => ({}))
  $: {
    if ($dimensionsChanged) {
      refreshTasks()
      refreshTimeRanges()
    }
  }

  setContext('dimensions', {
    from: _from,
    to: _to,
    width: _width,
    visibleWidth,
    visibleHeight,
    headerHeight,
    dimensionsChanged,
  })

  setContext('options', {
    taskContent,
    rowPadding: _rowPadding,
    rowHeight: _rowHeight,
    resizeHandleWidth: resizeHandleWidth,
    reflectOnParentRows,
    reflectOnChildRows,
    onTaskButtonClick,
  })

  const hoveredRow = writable()
  const selectedRow = writable()

  const ganttContext = {
    scrollables,
    hoveredRow,
    selectedRow,
  }
  setContext('gantt', ganttContext)

  onMount(() => {
    Object.assign(ganttContext, {
      rowContainer,
      mainContainer,
      mainHeaderContainer,
    })

    api.registerEvent('tasks', 'move')
    api.registerEvent('tasks', 'select')
    api.registerEvent('tasks', 'switchRow')
    api.registerEvent('tasks', 'moveEnd')
    api.registerEvent('tasks', 'change')
    api.registerEvent('tasks', 'changed')
    api.registerEvent('gantt', 'viewChanged')

    if (window.FileReader) {
      api.registerEvent('gantt', 'dragStart')
      api.registerEvent('gantt', 'dragOver')
      api.registerEvent('gantt', 'drop')
      api.registerEvent('gantt', 'dragLeave')
    }

    mounted = true
  })

  onDelegatedEvent('click', 'data-task-id', (event, data, target) => {
    const taskId = +data
    if (event.ctrlKey) {
      selectionManager.toggleSelection(taskId)
    } else {
      selectionManager.selectSingle(taskId)
    }
    api.tasks.raise.select($taskStore.entities[taskId])
  })

  onDelegatedEvent('mouseover', 'data-row-id', (event, data, target) => {
    $hoveredRow = +data
  })

  onDelegatedEvent('click', 'data-row-id', (event, data, target) => {
    $selectedRow = +data
  })

  onDestroy(() => {
    offDelegatedEvent('click', 'data-task-id')
    offDelegatedEvent('click', 'data-row-id')
  })

  let __scrollTop = 0
  let __scrollLeft = 0
  function scrollable(node) {
    const onscroll = (event) => {
      const { scrollTop, scrollLeft } = node

      scrollables.forEach((scrollable) => {
        if (scrollable.orientation === 'horizontal') {
          scrollable.node.scrollLeft = scrollLeft
        } else {
          scrollable.node.scrollTop = scrollTop
        }
      })

      __scrollTop = scrollTop
      __scrollLeft = scrollLeft
    }

    node.addEventListener('scroll', onscroll)
    return {
      destroy() {
        node.removeEventListener('scroll', onscroll, false)
      },
    }
  }

  function horizontalScrollListener(node) {
    scrollables.push({ node, orientation: 'horizontal' })
  }

  function onResize(event) {
    tableWidth = event.detail.left
  }

  let zoomLevel = 0
  let zooming = false
  async function onwheel(event) {
    if (event.ctrlKey) {
      event.preventDefault()

      const prevZoomLevel = zoomLevel
      if (event.deltaY > 0) {
        zoomLevel = Math.max(zoomLevel - 1, 0)
      } else {
        zoomLevel = Math.min(zoomLevel + 1, zoomLevels.length - 1)
      }

      if (prevZoomLevel != zoomLevel && zoomLevels[zoomLevel]) {
        const options = {
          columnUnit: columnUnit,
          columnOffset: columnOffset,
          minWidth: $_minWidth,
          ...zoomLevels[zoomLevel],
        }

        const scale = options.minWidth / $_width
        const node = mainContainer
        const mousepos = getRelativePos(node, event)
        const before = node.scrollLeft + mousepos.x
        const after = before * scale
        const scrollLeft = after - mousepos.x + node.clientWidth / 2

        console.log('scrollLeft', scrollLeft)

        columnUnit = options.columnUnit
        columnOffset = options.columnOffset
        $_minWidth = options.minWidth

        if (options.headers) headers = options.headers

        if (options.fitWidth) $_fitWidth = options.fitWidth

        api.gantt.raise.viewChanged()
        zooming = true
        await tick()
        node.scrollLeft = scrollLeft
        zooming = false
      }
    }
  }

  function onDateSelected(event) {
    $_from = event.detail.from.clone()
    $_to = event.detail.to.clone()
  }

  function onDragStart(event) {
    event.stopPropagation()
    event.preventDefault()
    // console.log('Gantt', 'dragstart', event)
    api.gantt.raise.dragStart()
  }
  function onDragOver(event) {
    event.stopPropagation()
    event.preventDefault()

    // console.log('Gantt', 'dragover', event)
    api.gantt.raise.dragOver()
  }
  function onDrop(event) {
    event.stopPropagation()
    event.preventDefault()

    // console.log('Gantt', 'drop', event)
    api.gantt.raise.drop()
  }
  function onDragLeave(event) {
    event.stopPropagation()
    event.preventDefault()

    // console.log('Gantt', 'dragleave', event)
    api.gantt.raise.dragLeave()
  }

  function initRows(rowsData) {
    const rows = rowFactory.createRows(rowsData)
    rowStore.addAll(rows)
  }

  async function initTasks(taskData) {
    await tick()

    const tasks = []
    const opts = { rowPadding: $_rowPadding }
    taskData.forEach((t) => {
      const task = taskFactory.createTask(t)
      const row = $rowStore.entities[task.model.resourceId]
      task.reflections = []

      if (reflectOnChildRows && row.allChildren) {
        row.allChildren.forEach((r) => {
          const reflectedTask = reflectTask(task, r, opts)
          task.reflections.push(reflectedTask.model.id)
          tasks.push(reflectedTask)
        })
      }

      if (reflectOnParentRows && row.allParents.length > 0) {
        row.allParents.forEach((r) => {
          const reflectedTask = reflectTask(task, r, opts)
          task.reflections.push(reflectedTask.model.id)
          tasks.push(reflectedTask)
        })
      }

      tasks.push(task)
    })
    taskStore.addAll(tasks)
  }

  function initTimeRanges(timeRangeData) {
    const timeRanges = timeRangeData.map((timeRange) => {
      return timeRangeFactory.create(timeRange)
    })
    timeRangeStore.addAll(timeRanges)
  }

  function onModuleInit(module) {}

  async function tickWithoutCSSTransition() {
    disableTransition = false
    await tick()
    ganttElement.offsetHeight // force a reflow
    disableTransition = true
  }

  export const api = new GanttApi()
  const selectionManager = new SelectionManager()

  export const taskFactory = new TaskFactory(columnService)
  $: {
    taskFactory.rowPadding = $_rowPadding
    taskFactory.rowEntities = $rowStore.entities
  }

  export const rowFactory = new RowFactory()
  $: rowFactory.rowHeight = rowHeight

  export const dndManager = new DragDropManager(rowStore)
  export const timeRangeFactory = new TimeRangeFactory(columnService)

  export const utils = new GanttUtils()
  $: {
    utils.from = $_from
    utils.to = $_to
    utils.width = $_width
    utils.magnetOffset = magnetOffset
    utils.magnetUnit = magnetUnit
  }

  setContext('services', {
    utils,
    api,
    dndManager,
    selectionManager,
    columnService,
  })

  export function refreshTimeRanges() {
    timeRangeStore._update(({ ids, entities }) => {
      ids.forEach((id) => {
        const timeRange = entities[id]
        const newLeft =
          columnService.getPositionByDate(timeRange.model.from) | 0
        const newRight = columnService.getPositionByDate(timeRange.model.to) | 0

        timeRange.left = newLeft
        timeRange.width = newRight - newLeft
      })
      return { ids, entities }
    })
  }

  export function refreshTasks() {
    $allTasks.forEach((task) => {
      const newLeft = columnService.getPositionByDate(task.model.from) | 0
      const newRight = columnService.getPositionByDate(task.model.to) | 0

      task.left = newLeft
      task.width = newRight - newLeft
    })

    taskStore.refresh()
  }

  export function getRowContainer() {
    return rowContainer
  }

  export function selectTask(id) {
    const task = $taskStore.entities[id]
    if (task) {
      selectionManager.selectSingle(task)
    }
  }

  export function unselectTasks() {
    selectionManager.clearSelection()
  }

  export function scrollToRow(id, scrollBehavior = 'auto') {
    const { scrollTop, clientHeight } = mainContainer

    const index = $allRows.findIndex((r) => r.model.id == id)
    if (index === -1) return
    const targetTop = index * rowHeight

    if (targetTop < scrollTop) {
      mainContainer.scrollTo({
        top: targetTop,
        behavior: scrollBehavior,
      })
    }

    if (targetTop > scrollTop + clientHeight) {
      mainContainer.scrollTo({
        top: targetTop + rowHeight - clientHeight,
        behavior: scrollBehavior,
      })
    }
  }

  export function scrollToTask(id, scrollBehavior = 'auto') {
    const { scrollLeft, scrollTop, clientWidth, clientHeight } = mainContainer

    const task = $taskStore.entities[id]
    if (!task) return
    const targetLeft = task.left
    const rowIndex = $allRows.findIndex(
      (r) => r.model.id == task.model.resourceId,
    )
    const targetTop = rowIndex * rowHeight

    const options = {
      top: undefined,
      left: undefined,
      behavior: scrollBehavior,
    }

    if (targetLeft < scrollLeft) {
      options.left = targetLeft
    }

    if (targetLeft > scrollLeft + clientWidth) {
      options.left = targetLeft + task.width - clientWidth
    }

    if (targetTop < scrollTop) {
      options.top = targetTop
    }

    if (targetTop > scrollTop + clientHeight) {
      options.top = targetTop + rowHeight - clientHeight
    }

    mainContainer.scrollTo(options)
  }

  export function updateTask(model) {
    const task = taskFactory.createTask(model)
    taskStore.upsert(task)
  }

  export function updateTasks(taskModels) {
    const tasks = taskModels.map((model) => taskFactory.createTask(model))
    taskStore.upsertAll(tasks)
  }

  export function updateRow(model) {
    const row = rowFactory.createRow(model)
    rowStore.upsert(row)
  }

  export function updateRows(rowModels) {
    const rows = rowModels.map((model) => rowFactory.createRow(model))
    rowStore.upsertAll(rows)
  }

  export function getRow(resourceId) {
    return $rowStore.entities[resourceId]
  }

  export function getTask(id) {
    return $taskStore.entities[id]
  }

  export function getTasks(resourceId) {
    if ($rowTaskCache[resourceId]) {
      return $rowTaskCache[resourceId].map((id) => $taskStore.entities[id])
    }
    return null
  }

  let filteredRows = []
  $: filteredRows = $allRows.filter((row) => !row.hidden)

  let rightScrollbarVisible
  $: rightScrollbarVisible = rowContainerHeight > $visibleHeight

  let rowContainerHeight
  $: rowContainerHeight = filteredRows.length * rowHeight

  let startIndex
  $: startIndex = Math.floor(__scrollTop / rowHeight)

  let endIndex
  $: endIndex = Math.min(
    startIndex + Math.ceil($visibleHeight / rowHeight),
    filteredRows.length - 1,
  )

  let paddingTop = 0
  $: paddingTop = startIndex * rowHeight

  let paddingBottom = 0
  $: paddingBottom = (filteredRows.length - endIndex - 1) * rowHeight

  let visibleRows = []
  $: visibleRows = filteredRows.slice(startIndex, endIndex + 1)

  let visibleTasks
  $: {
    const tasks = []
    visibleRows.forEach((row) => {
      if ($rowTaskCache[row.model.id]) {
        $rowTaskCache[row.model.id].forEach((id) => {
          tasks.push($taskStore.entities[id])
        })
      }
    })
    visibleTasks = tasks
  }

  let noVisibleRows = false
  $: noVisibleRows = visibleRows.length == 0

  let disableTransition = true
  $: if ($dimensionsChanged) tickWithoutCSSTransition()
</script>

<style>
  .sg-disable-transition :global(.sg-task),
  .sg-disable-transition :global(.sg-milestone) {
    transition: transform 0s, background-color 0.2s, width 0s !important;
  }

  :global(.sg-view:not(:first-child)) {
    margin-left: 5px;
  }

  /* This class should take into account varying widths of the scroll bar */
  .right-scrollbar-visible {
    padding-right: 17px;
  }

  .sg-timeline {
    flex: 1 1 0%;
    display: flex;
    flex-direction: column;
    overflow-x: auto;
  }

  .sg-gantt {
    display: flex;

    width: 100%;
    height: 100%;
    position: relative;
  }

  .sg-foreground {
    box-sizing: border-box;
    overflow: hidden;
    top: 0;
    left: 0;
    position: absolute;
    width: 100%;
    height: 100%;
    z-index: 1;
    pointer-events: none;
  }

  .sg-rows {
    width: 100%;
    box-sizing: border-box;
    overflow: hidden;
  }

  .sg-timeline-body {
    overflow: auto;
    flex: 1 1 auto;
  }

  .sg-header {
  }

  .header-container {
  }

  .sg-header-scroller {
    border-right: 1px solid #efefef;
    overflow: hidden;
    position: relative;
  }

  .content {
    position: relative;
  }

  :global(*) {
    box-sizing: border-box;
  }
</style>

<div
  class="sg-gantt {classes}"
  class:sg-disable-transition={!disableTransition}
  draggable="true"
  bind:this={ganttElement}
  on:click={onEvent}
  on:mouseover={onEvent}>
  <form
    method="post"
    action=""
    enctype="multipart/form-data"
    bind:this={ganttDragElement}>
    {#each ganttTableModules as module}
      <svelte:component
        this={module}
        {rowContainerHeight}
        {paddingTop}
        {paddingBottom}
        {tableWidth}
        {...$$restProps}
        on:init={onModuleInit}
        {visibleRows} />

      <!-- <Resizer x={tableWidth} on:resize={onResize} container={ganttElement} /> -->
    {/each}
    <div
      class="sg-timeline sg-view rows-count-{visibleRows.length}"
      class:sg-timeline-rows-0={noVisibleRows}>
      <div
        class="sg-header"
        bind:this={mainHeaderContainer}
        bind:clientHeight={$headerHeight}
        class:right-scrollbar-visible={rightScrollbarVisible}>
        <div class="sg-header-scroller" use:horizontalScrollListener>
          <div class="header-container" style="width:{$_width}px">
            <ColumnHeader
              {headers}
              {columnUnit}
              {columnOffset}
              on:dateSelected={onDateSelected} />
            {#each $allTimeRanges as timeRange (timeRange.id)}
              <TimeRangeHeader {...timeRange} />
            {/each}
          </div>
        </div>
      </div>

      <div
        class="sg-timeline-body"
        bind:this={mainContainer}
        use:scrollable
        class:zooming
        on:wheel={onwheel}
        bind:clientHeight={$visibleHeight}
        bind:clientWidth={$visibleWidth}>
        <div class="content" style="width:{$_width}px">
          <Columns {columns} {columnStrokeColor} {columnStrokeWidth} />
          <div
            class="sg-rows"
            bind:this={rowContainer}
            style="height:{rowContainerHeight}px;">
            <div style="transform: translateY({paddingTop}px);">
              {#each visibleRows as row (row.model.id)}
                <Row {row} />
              {/each}
            </div>
          </div>
          <div class="sg-foreground">
            {#each $allTimeRanges as timeRange (timeRange.id)}
              <TimeRange {...timeRange} />
            {/each}

            {#each visibleTasks as task (task.model.id)}
              <Task
                model={task.model}
                left={task.left}
                width={task.width}
                height={task.height}
                top={task.top}
                {...task} />
            {/each}
          </div>
          {#each ganttBodyModules as module}
            <svelte:component
              this={module}
              {paddingTop}
              {paddingBottom}
              {visibleRows}
              {...$$restProps}
              on:init={onModuleInit} />
          {/each}
        </div>
      </div>
    </div>
    <input
      type="file"
      id="fileDragToShare"
      style="display: none"
      name="fileSelect[]" />
    <input type="submit" style="display: none" />
  </form>
  <div class="sg-table sg-table-last sg-view">
    <div class="sg-table-header sg-table-header-last">
      <div class="sg-table-header-cell sg-table-cell sg-table-header-cell-last">
        <div class="btn go-next-day">
          <i class="fas fa-angle-right" />
        </div>
        <div class="btn go-next-month">
          <i class="fas fa-angle-double-right" />
        </div>
      </div>
    </div>
  </div>
</div>
