<script>
  let rows = 12;
  let cols = 12;
  let cellSize = 16;
  let cellGap = 8;
  let cornerRadius = 4;
  let fillColor = "#2B323B";

  let activeCells = new Map();
  let isPainting = false;
  let paintValue = true;
  let copyStatus = "";
  let gridWidth = 0;
  let gridHeight = 0;
  let activeCount = 0;
  let svgOutput = "";
  let isResizing = false;
  let resizeStartX = 0;
  let resizeStartY = 0;
  let resizeStartSize = 0;
  let radiusFactor = 50;
  let isGridResizing = false;
  let gridResizeStartX = 0;
  let gridResizeStartY = 0;
  let gridResizeStartRows = 0;
  let gridResizeStartCols = 0;
  let isGapDragging = false;
  let gapStartX = 0;
  let gapStartValue = 0;

  const keyFor = (row, col) => `${row}:${col}`;

  const clampInt = (value, min, max = Number.POSITIVE_INFINITY) => {
    const parsed = Number.parseInt(value, 10);
    if (Number.isNaN(parsed)) {
      return min;
    }
    return Math.min(max, Math.max(min, parsed));
  };

  const normalizeInputs = () => {
    rows = clampInt(rows, 1, 200);
    cols = clampInt(cols, 1, 200);
    cellSize = clampInt(cellSize, 1, 96);
    cellGap = clampInt(cellGap, 0, 256);
    radiusFactor = clampInt(radiusFactor, 0, 100);
  };

  const pruneActiveCells = (cells, maxRows, maxCols) => {
    let changed = false;
    const next = new Map();
    for (const [entry, color] of cells) {
      const [rowText, colText] = entry.split(":");
      const row = Number(rowText);
      const col = Number(colText);
      if (row >= 0 && row < maxRows && col >= 0 && col < maxCols) {
        next.set(entry, color);
      } else {
        changed = true;
      }
    }
    return changed ? next : cells;
  };

  const setCell = (row, col, value) => {
    const next = new Map(activeCells);
    const key = keyFor(row, col);
    if (value) {
      next.set(key, fillColor);
    } else {
      next.delete(key);
    }
    activeCells = next;
  };

  const getCellFromEvent = (event) => {
    const target = event.target;
    if (!(target instanceof HTMLElement)) {
      return null;
    }
    const row = target.dataset.row;
    const col = target.dataset.col;
    if (row === undefined || col === undefined) {
      return null;
    }
    return { row: Number(row), col: Number(col) };
  };

  const startPainting = (event) => {
    const cell = getCellFromEvent(event);
    if (!cell) {
      return;
    }
    const isActive = activeCells.has(keyFor(cell.row, cell.col));
    paintValue = !isActive;
    isPainting = true;
    setCell(cell.row, cell.col, paintValue);
  };

  const continuePainting = (event) => {
    if (!isPainting) {
      return;
    }
    const cell = getCellFromEvent(event);
    if (!cell) {
      return;
    }
    setCell(cell.row, cell.col, paintValue);
  };

  const stopPainting = () => {
    isPainting = false;
  };

  const startResize = (event) => {
    event.preventDefault();
    isResizing = true;
    resizeStartX = event.clientX;
    resizeStartY = event.clientY;
    resizeStartSize = cellSize;
    if (event.currentTarget instanceof HTMLElement && event.currentTarget.setPointerCapture) {
      event.currentTarget.setPointerCapture(event.pointerId);
    }
  };

  const handleResizeMove = (event) => {
    if (!isResizing) {
      return;
    }
    const delta = Math.max(event.clientX - resizeStartX, event.clientY - resizeStartY);
    cellSize = clampInt(Math.round(resizeStartSize + delta), 1, 96);
  };

  const stopResize = () => {
    isResizing = false;
  };

  const startGridResize = (event) => {
    event.preventDefault();
    isGridResizing = true;
    gridResizeStartX = event.clientX;
    gridResizeStartY = event.clientY;
    gridResizeStartRows = rows;
    gridResizeStartCols = cols;
    if (event.currentTarget instanceof HTMLElement && event.currentTarget.setPointerCapture) {
      event.currentTarget.setPointerCapture(event.pointerId);
    }
  };

  const handleGridResizeMove = (event) => {
    if (!isGridResizing) {
      return;
    }
    const step = Math.max(1, cellSize + cellGap);
    const deltaX = event.clientX - gridResizeStartX;
    const deltaY = event.clientY - gridResizeStartY;
    const nextCols = clampInt(gridResizeStartCols + Math.round(deltaX / step), 1, 200);
    const nextRows = clampInt(gridResizeStartRows + Math.round(deltaY / step), 1, 200);
    cols = nextCols;
    rows = nextRows;
  };

  const stopGridResize = () => {
    isGridResizing = false;
  };

  const startGapDrag = (event) => {
    event.preventDefault();
    isGapDragging = true;
    gapStartX = event.clientX;
    gapStartValue = cellGap;
    if (event.currentTarget instanceof HTMLElement && event.currentTarget.setPointerCapture) {
      event.currentTarget.setPointerCapture(event.pointerId);
    }
  };

  const handleGapDragMove = (event) => {
    if (!isGapDragging) {
      return;
    }
    const delta = event.clientX - gapStartX;
    cellGap = clampInt(gapStartValue + Math.round(delta), 0, 256);
  };

  const stopGapDrag = () => {
    isGapDragging = false;
  };

  const stopPointerActions = () => {
    stopPainting();
    stopResize();
    stopGridResize();
    stopGapDrag();
  };

  const handlePointerMove = (event) => {
    handleResizeMove(event);
    handleGridResizeMove(event);
    handleGapDragMove(event);
  };

  const clearGrid = () => {
    activeCells = new Map();
  };


  const copySvg = async () => {
    try {
      await navigator.clipboard.writeText(svgOutput);
      copyStatus = "Copied!";
    } catch (error) {
      copyStatus = "Copy failed";
    }
    setTimeout(() => {
      copyStatus = "";
    }, 1500);
  };

  const downloadSvg = () => {
    const blob = new Blob([svgOutput], { type: "image/svg+xml;charset=utf-8" });
    const url = URL.createObjectURL(blob);
    const link = document.createElement("a");
    link.href = url;
    link.download = "grid-drawing.svg";
    link.click();
    URL.revokeObjectURL(url);
  };

  $: normalizeInputs();
  $: cornerRadius = Math.round((cellSize / 2) * (radiusFactor / 100));
  $: activeCells = pruneActiveCells(activeCells, rows, cols);
  $: gridWidth = cols * cellSize + (cols - 1) * cellGap;
  $: gridHeight = rows * cellSize + (rows - 1) * cellGap;
  $: totalCells = rows * cols;
  $: activeCount = activeCells.size;
  $: previewCellSize = Math.max(1, Math.min(cellSize, 96));
  $: previewRadius = Math.min(cornerRadius, previewCellSize / 2);
  $: svgOutput = generateSvg(
    activeCells,
    gridWidth,
    gridHeight,
    cellSize,
    cellGap,
    cornerRadius,
    fillColor
  );

  function generateSvg(cells, width, height, size, gap, radius, color) {
    const lines = [];
    for (const [entry, entryColor] of cells) {
      const [rowText, colText] = entry.split(":");
      const row = Number(rowText);
      const col = Number(colText);
      const x = col * (size + gap);
      const y = row * (size + gap);
      lines.push(
        `<rect x="${x}" y="${y}" width="${size}" height="${size}" rx="${radius}" fill="${entryColor || color}" />`
      );
    }

    const rects = lines.length ? `  ${lines.join("\n  ")}\n` : "";
    return `<svg width="${width}" height="${height}" viewBox="0 0 ${width} ${height}" xmlns="http://www.w3.org/2000/svg">\n${rects}</svg>`;
  }
</script>

<svelte:window on:pointerup={stopPointerActions} on:pointercancel={stopPointerActions} on:pointermove={handlePointerMove} />

<main class="page">
  <header class="header">
    <div>
      <h1>Grid SVG Drawer</h1>
      <p>Click or drag to paint cells, then export the SVG.</p>
    </div>
  </header>

  <div class="actions-toolbar">
    <button type="button" on:click={clearGrid}>Clear grid</button>
    <div class="summary">
      <span>{rows} x {cols}</span>
      <span>{activeCount} out of {totalCells} cells filled</span>
      <span>{gridWidth} x {gridHeight}px</span>
      <span>Cell {cellSize}px</span>
      <span>Gap {cellGap}px</span>
      <span>Radius {cornerRadius}px</span>
    </div>
    {#if copyStatus}
      <span class="status">{copyStatus}</span>
    {/if}
  </div>

  <section class="panel">
    <div class="controls">
      <div class="visualizer">
        <label class="radius-slider">
          <span class="radius-end square" aria-hidden="true"></span>
          <span class="sr-only">Corner radius</span>
          <input
            type="range"
            min="0"
            max="100"
            value={radiusFactor}
            on:input={(event) => (radiusFactor = clampInt(event.currentTarget.value, 0, 100))}
          />
          <span class="radius-end circle" aria-hidden="true"></span>
        </label>
        <div class="visualizer-preview" style={`gap: ${cellGap}px;`}>
          <div
            class="visualizer-cell is-swatch"
            aria-label="Fill color"
            style={`width: ${previewCellSize}px; height: ${previewCellSize}px; border-radius: ${previewRadius}px; --preview-color: ${fillColor};`}
          >
            <input
              class="color-picker-input"
              type="color"
              value={fillColor}
              aria-label="Fill color"
              on:input={(event) => (fillColor = event.currentTarget.value)}
            />
            <span class="resize-handle" on:pointerdown|stopPropagation={startResize}></span>
          </div>
          <div
            class="visualizer-cell gap-handle"
            aria-label="Grid gap"
            style={`width: ${previewCellSize}px; height: ${previewCellSize}px; border-radius: ${previewRadius}px;`}
            on:pointerdown|stopPropagation={startGapDrag}
          ></div>
        </div>
      </div>
      <div class="actions-panel"></div>
    </div>

    <div class="grid-panel">
      <div class="grid-resize-wrapper">
        <div
          class="grid"
          style={`grid-template-columns: repeat(${cols}, ${cellSize}px); grid-auto-rows: ${cellSize}px; gap: ${cellGap}px; --fill-color: ${fillColor}; --corner-radius: ${cornerRadius}px;`}
          on:pointerdown|preventDefault={startPainting}
          on:pointermove={continuePainting}
          role="grid"
          aria-label="Drawing grid"
        >
          {#each Array.from({ length: rows }) as _, row}
            {#each Array.from({ length: cols }) as _, col}
              {@const cellKey = keyFor(row, col)}
              {@const cellColor = activeCells.get(cellKey)}
              <button
                type="button"
                class:active={activeCells.has(cellKey)}
                data-row={row}
                data-col={col}
                aria-label={`Cell ${row + 1}, ${col + 1}`}
                style={cellColor ? `background: ${cellColor}; border-color: ${cellColor};` : ""}
              ></button>
            {/each}
          {/each}
        </div>
        <span
          class="grid-resize-handle"
          aria-label="Resize grid"
          on:pointerdown|stopPropagation={startGridResize}
        ></span>
      </div>
    </div>
  </section>

  <section class="panel export">
    <div class="export-actions">
      <button type="button" on:click={copySvg}>Copy SVG</button>
      <button type="button" on:click={downloadSvg}>Download SVG</button>
      {#if copyStatus}
        <span class="status">{copyStatus}</span>
      {/if}
    </div>
    <details>
      <summary>SVG output</summary>
      <textarea readonly rows="12" bind:value={svgOutput}></textarea>
    </details>
  </section>
</main>

<style>
  :global(body) {
    margin: 0;
    font-family: "Inter", system-ui, sans-serif;
    color: #1f2933;
    background: #f7f9fb;
  }

  .page {
    display: flex;
    flex-direction: column;
    gap: 2rem;
    padding: 2.5rem clamp(1.5rem, 4vw, 3rem);
  }

  .header {
    display: flex;
    flex-wrap: wrap;
    gap: 1.5rem;
    align-items: center;
    justify-content: space-between;
  }

  .header h1 {
    margin: 0 0 0.4rem;
    font-size: 2rem;
  }

  .header p {
    margin: 0;
    color: #54616f;
  }

  .summary {
    display: flex;
    gap: 1rem;
    flex-wrap: wrap;
    font-size: 0.95rem;
    color: #334155;
  }

  .panel {
    background: #ffffff;
    border-radius: 1.25rem;
    padding: 1.5rem;
    box-shadow: 0 20px 45px rgba(15, 23, 42, 0.08);
  }

  .controls {
    display: flex;
    flex-wrap: wrap;
    gap: 2rem;
    align-items: flex-start;
  }

  .visualizer {
    display: flex;
    gap: 1.5rem;
    flex-direction: column;
    align-items: flex-start;
  }

  .visualizer-preview {
    display: flex;
    flex-direction: row;
    gap: 0.75rem;
    align-items: center;
    padding: 1rem;
    border-radius: 1rem;
    border: 1px solid #e2e8f0;
    background: #f8fafc;
  }

  .visualizer-cell {
    position: relative;
    border: 1px solid #d9e2ec;
    background: white;
    box-sizing: border-box;
  }

  .visualizer-preview .is-swatch {
    background: var(--preview-color, #2b323b);
    border-color: var(--preview-color, #2b323b);
    position: relative;
    cursor: pointer;
  }

  .gap-handle {
    background: white;
    border-color: #d9e2ec;
    cursor: ew-resize;
  }

  .visualizer-preview .color-picker-input {
    position: absolute;
    inset: 0;
    opacity: 0;
    cursor: pointer;
    z-index: 1;
  }

  .radius-slider {
    width: max(var(--preview-cell-size), 50px);
    height: var(--preview-cell-size);
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;
    line-height: 0;
    gap: 2px;
  }

  .radius-slider input[type="range"] {
    width: calc(max(var(--preview-cell-size), 50px) - 20px);
    height: 1px;
    margin: 0;
    background: transparent;
    -webkit-appearance: none;
    appearance: none;
    cursor: ew-resize;
  }

  .radius-slider input[type="range"]::-webkit-slider-runnable-track {
    height: 1px;
    background: #4c6ef5;
    border-radius: 999px;
  }

  .radius-slider input[type="range"]::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 10px;
    height: 10px;
    border-radius: 999px;
    background: #4c6ef5;
    margin-top: -4.5px;
  }

  .radius-slider input[type="range"]::-moz-range-track {
    height: 1px;
    background: #4c6ef5;
    border-radius: 999px;
  }

  .radius-slider input[type="range"]::-moz-range-thumb {
    width: 10px;
    height: 10px;
    border-radius: 999px;
    background: #4c6ef5;
    border: none;
  }

  .radius-end {
    width: 8px;
    height: 8px;
    background: #000000;
    flex: 0 0 auto;
  }

  .radius-end.circle {
    border-radius: 999px;
  }

  .resize-handle {
    position: absolute;
    right: -16px;
    bottom: -16px;
    width: 14px;
    height: 14px;
    border-radius: 4px;
    background: #4c6ef5;
    border: 2px solid white;
    box-shadow: 0 4px 8px rgba(15, 23, 42, 0.25);
    cursor: nwse-resize;
    z-index: 2;
  }

  .grid-resize-wrapper {
    position: relative;
    width: fit-content;
  }

  .grid-resize-handle {
    position: absolute;
    right: -18px;
    bottom: -18px;
    width: 16px;
    height: 16px;
    border-radius: 4px;
    background: #4c6ef5;
    border: 2px solid white;
    box-shadow: 0 4px 8px rgba(15, 23, 42, 0.25);
    cursor: nwse-resize;
  }

  input[type="range"] {
    accent-color: #4c6ef5;
  }


  .actions-panel {
    display: none;
  }

  .actions-toolbar {
    display: flex;
    gap: 0.75rem;
    align-items: center;
    margin: 1rem 0 0.5rem;
    flex-wrap: wrap;
  }

  .actions-toolbar button {
    border: none;
    border-radius: 999px;
    padding: 0.55rem 1rem;
    background: #2b323b;
    color: white;
    font-size: 0.9rem;
    cursor: pointer;
  }

  .actions-toolbar button:first-child {
    background: #e03131;
  }

  .actions-toolbar button:nth-child(2) {
    background: #12b886;
  }

  .actions-toolbar button:nth-child(3) {
    background: #4c6ef5;
  }

  .export-actions {
    display: flex;
    gap: 0.75rem;
    align-items: center;
    margin-bottom: 0.75rem;
  }

  .export-actions button {
    border: none;
    border-radius: 999px;
    padding: 0.55rem 1rem;
    background: #2b323b;
    color: white;
    font-size: 0.9rem;
    cursor: pointer;
  }

  .export-actions button:first-child {
    background: #12b886;
  }

  .export-actions button:nth-child(2) {
    background: #4c6ef5;
  }

  details summary {
    cursor: pointer;
    font-weight: 600;
    color: #1f2933;
    padding: 0.35rem 0;
    list-style: none;
  }

  details summary::before {
    content: "▸";
    display: inline-block;
    margin-right: 0.5rem;
    transition: transform 0.2s ease;
  }

  details[open] summary::before {
    transform: rotate(90deg);
  }

  details summary::-webkit-details-marker {
    display: none;
  }

  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
  }

  label {
    display: flex;
    flex-direction: column;
    gap: 0.35rem;
    font-size: 0.9rem;
    color: #2b323b;
  }

  input[type="color"] {
    width: 100%;
    border: 1px solid #d0d7de;
    border-radius: 0.6rem;
    padding: 0.3rem;
    height: 2.6rem;
  }
  
  .status {
    font-size: 0.85rem;
    color: #16a34a;
  }

  .grid-panel {
    margin-top: 1.5rem;
    overflow: visible;
    padding: 1rem;
    border-radius: 1rem;
    border: 1px solid #e2e8f0;
    background: #f8fafc;
    width: fit-content;
  }

  .grid {
    display: grid;
    width: fit-content;
  }

  .grid button {
    width: 100%;
    height: 100%;
    border-radius: var(--corner-radius, 0.35rem);
    border: 1px solid #d9e2ec;
    background: white;
    cursor: pointer;
    padding: 0;
  }

  .grid button.active {
    background: var(--fill-color, #2b323b);
    border-color: var(--fill-color, #2b323b);
  }

  .export {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
  }

  textarea {
    width: 100%;
    border-radius: 0.8rem;
    border: 1px solid #d0d7de;
    padding: 0.75rem;
    font-family: "SFMono-Regular", ui-monospace, SFMono-Regular, Menlo, monospace;
    font-size: 0.85rem;
    min-height: 220px;
    resize: vertical;
  }
  </style>
