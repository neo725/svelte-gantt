<script>
  import { createEventDispatcher, onMount } from 'svelte'

  const dispatch = createEventDispatcher()

  import { Draggable } from 'src/core/drag'
  import { setCursor } from 'src/utils/domUtils'

  export let x = 240
  export let container

  let dragging = false
  const dragOptions = {
    onDrag: (event) => {
      ;(x = event.x), (dragging = true)
      dispatch('resize', { left: x })
      setCursor('col-resize')
    },
    onDrop: (event) => {
      ;(x = event.x), (dragging = false)
      dispatch('resize', { left: x })
      setCursor('default')
    },
    dragAllowed: true,
    resizeAllowed: false,
    container: container,
    getX: () => x,
    getY: () => 0,
    getWidth: () => 0,
  }

  $: dragOptions.container = container

  function resizer(node) {
    return new Draggable(node, dragOptions)
  }
</script>

<style>
  .sg-resize {
    display: none;
    z-index: 2;
    background: #e9eaeb;
    width: 0;
    cursor: col-resize;
    position: absolute;
    height: 100%;

    transition: width 0.2s, transform 0.2s;
  }

  .sg-resize:hover {
    transform: translateX(-2px);
    width: 10px;
  }
</style>

<div class="sg-resize" style="left:{x}px" use:resizer />
