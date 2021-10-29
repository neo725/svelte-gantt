<script>
  import { createEventDispatcher, getContext } from 'svelte'

  const dispatch = createEventDispatcher()

  import { duration as momentDuration } from 'moment'

  const { from, to, width } = getContext('dimensions')

  export let header
  export let baseWidth
  export let baseDuration

  export let columnWidth
  $: {
    const offset = header.offset || 1
    const duration = momentDuration(offset, header.unit).asMilliseconds()
    const ratio = duration / baseDuration
    columnWidth = baseWidth * ratio
  }

  export let columnCount
  $: {
    columnCount = Math.ceil($width / columnWidth)
    if (!isFinite(columnCount)) {
      console.error('columnCount is not finite')
      columnCount = 0
    }
  }

  let _headers = []
  $: {
    const headers = []
    let headerTime = $from.clone().startOf(header.unit)
    const offset = header.offset || 1

    for (let i = 0; i < columnCount; i++) {
      let _header = {
        width: Math.min(columnWidth, $width),
        label: headerTime.format(header.format),
        from: headerTime.clone(),
        to: headerTime.clone().add(offset, header.unit),
        unit: header.unit,
        classes: header.classes,
      }

      let isMarkToday = header['today'] || false

      if (isMarkToday && _header.from.isSame(new Date(), 'day')) {
        _header.label = 'TODAY'

        let _classes = _header['classes'] || ''
        if (_classes.length > 0) {
          _classes += ' '
        }
        _classes += 'sg-table-header-cell-today'
        _header.classes = _classes
      }

      headers.push(_header)

      headerTime.add(offset, header.unit)
    }
    _headers = headers
  }
</script>

<style>
  .column-header-row {
    box-sizing: border-box;
    white-space: nowrap;
    height: 32px;
  }

  .column-header-cell {
    display: inline-block;
    height: 100%;
    box-sizing: border-box;
    text-overflow: clip;
    /* vertical-align: top; */
    text-align: center;

    display: inline-flex;
    justify-content: center;
    align-items: center;
    font-size: 1em;
    font-size: 14px;
    font-weight: 300;
    transition: background 0.2s;

    /* cursor: pointer; */
    user-select: none;

    border-right: #efefef 1px solid;
    border-bottom: #efefef 1px solid;
  }

  /* .column-header-cell:hover {
    background: #f9f9f9;
  } */

  .column-header-cell.sticky > .column-header-cell-label {
    position: sticky;
    left: 1rem;
  }
</style>

<div class="column-header-row">
  {#each _headers as _header}
    <div
      class="column-header-cell {_header.classes || ''}"
      class:sticky={_header.sticky}
      style="width:{_header.width}px">
      <div class="column-header-cell-label">{_header.label || 'N/A'}</div>
    </div>
  {/each}
</div>
