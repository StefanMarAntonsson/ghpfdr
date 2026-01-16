<script>
  let rows = 12;
  let cols = 36;
  let cellSize = 16;
  let cellGap = 8;
  let cornerRadius = 4;
  let fillColor = "#2B323B";

  let activeCells = new Set();
  let isPainting = false;
  let paintValue = true;
  let copyStatus = "";
  let randomCount = 0;
  let gridWidth = 0;
  let gridHeight = 0;
  let activeCount = 0;
  let svgOutput = "";

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
    cellSize = clampInt(cellSize, 4, 64);
    cellGap = clampInt(cellGap, 0, 64);
    cornerRadius = clampInt(cornerRadius, 0, 256);
  };

  const pruneActiveCells = (cells, maxRows, maxCols) => {
    let changed = false;
    const next = new Set();
    for (const entry of cells) {
      const [rowText, colText] = entry.split(":");
      const row = Number(rowText);
      const col = Number(colText);
      if (row >= 0 && row < maxRows && col >= 0 && col < maxCols) {
        next.add(entry);
      } else {
        changed = true;
      }
    }
    return changed ? next : cells;
  };

  const setCell = (row, col, value) => {
    const next = new Set(activeCells);
    const key = keyFor(row, col);
    if (value) {
      next.add(key);
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

  const clearGrid = () => {
    activeCells = new Set();
  };

  const randomFill = () => {
    const total = rows * cols;
    if (total <= 0) {
      return;
    }
    const requested = clampInt(randomCount, 0, total);
    const targetCount =
      requested === 0
        ? Math.floor(Math.random() * Math.max(1, Math.ceil(total * 0.2))) + 1
        : requested;
    const createRandomSet = () => {
      const allKeys = [];
      for (let row = 0; row < rows; row += 1) {
        for (let col = 0; col < cols; col += 1) {
          allKeys.push(keyFor(row, col));
        }
      }
      for (let i = allKeys.length - 1; i > 0; i -= 1) {
        const swapIndex = Math.floor(Math.random() * (i + 1));
        [allKeys[i], allKeys[swapIndex]] = [allKeys[swapIndex], allKeys[i]];
      }
      return new Set(allKeys.slice(0, targetCount));
    };

    let next = createRandomSet();
    if (activeCells.size === targetCount && total > 1) {
      let attempts = 0;
      while (attempts < 5) {
        const previous = activeCells;
        next = createRandomSet();
        let isSame = previous.size === next.size;
        if (isSame) {
          for (const entry of previous) {
            if (!next.has(entry)) {
              isSame = false;
              break;
            }
          }
        }
        if (!isSame) {
          break;
        }
        attempts += 1;
      }
    }
    activeCells = next;
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
  $: activeCells = pruneActiveCells(activeCells, rows, cols);
  $: gridWidth = cols * cellSize + (cols - 1) * cellGap;
  $: gridHeight = rows * cellSize + (rows - 1) * cellGap;
  $: activeCount = activeCells.size;
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
    for (const entry of cells) {
      const [rowText, colText] = entry.split(":");
      const row = Number(rowText);
      const col = Number(colText);
      const x = col * (size + gap);
      const y = row * (size + gap);
      lines.push(
        `<rect x="${x}" y="${y}" width="${size}" height="${size}" rx="${radius}" fill="${color}" />`
      );
    }

    const rects = lines.length ? `  ${lines.join("\n  ")}\n` : "";
    return `<svg width="${width}" height="${height}" viewBox="0 0 ${width} ${height}" xmlns="http://www.w3.org/2000/svg">\n${rects}</svg>`;
  }
</script>

<svelte:window on:pointerup={stopPainting} on:pointercancel={stopPainting} />

<main class="page">
  <header class="header">
    <div>
      <h1>Grid SVG Drawer</h1>
      <p>Click or drag to paint cells, then export the SVG.</p>
    </div>
    <div class="summary">
      <span>{rows} x {cols}</span>
      <span>{activeCount} cells filled</span>
      <span>{gridWidth} x {gridHeight}px</span>
    </div>
  </header>

  <section class="panel">
    <div class="controls">
      <div class="controls-column">
        <div class="input-row">
          <label>
            Rows
            <input
              type="number"
              min="1"
              max="200"
              value={rows}
              on:input={(event) => (rows = clampInt(event.currentTarget.value, 1, 200))}
            />
          </label>
          <label>
            Columns
            <input
              type="number"
              min="1"
              max="200"
              value={cols}
              on:input={(event) => (cols = clampInt(event.currentTarget.value, 1, 200))}
            />
          </label>
        </div>
        <div class="input-row">
          <label>
            Cell size
            <input
              type="number"
              min="4"
              max="64"
              value={cellSize}
              on:input={(event) => (cellSize = clampInt(event.currentTarget.value, 4, 64))}
            />
          </label>
          <label>
            Cell gap
            <input
              type="number"
              min="0"
              max="64"
              value={cellGap}
              on:input={(event) => (cellGap = clampInt(event.currentTarget.value, 0, 64))}
            />
          </label>
        </div>
      </div>
      <div class="controls-column">
        <div class="input-row">
          <label>
            Corner radius
            <input
              type="number"
              min="0"
              max="256"
              value={cornerRadius}
              on:input={(event) => (cornerRadius = clampInt(event.currentTarget.value, 0, 256))}
            />
          </label>
          <label>
            Random cells
            <input
              type="number"
              min="0"
              max={rows * cols}
              value={randomCount}
              on:input={(event) => (randomCount = clampInt(event.currentTarget.value, 0, rows * cols))}
            />
          </label>
        </div>
        <div class="input-row">
          <label class="color">
            Fill color
            <div class="color-row">
              <input
                type="color"
                value={fillColor}
                on:input={(event) => (fillColor = event.currentTarget.value)}
              />
              <input
                type="text"
                value={fillColor}
                on:input={(event) => (fillColor = event.currentTarget.value)}
              />
            </div>
          </label>
        </div>
      </div>
      <div class="actions">
        <button type="button" on:click={clearGrid}>Clear grid</button>
        <button type="button" on:click={randomFill}>Random fill</button>
        <button type="button" on:click={copySvg}>Copy SVG</button>
        <button type="button" on:click={downloadSvg}>Download SVG</button>
        {#if copyStatus}
          <span class="status">{copyStatus}</span>
        {/if}
      </div>
    </div>

    <div class="grid-panel">
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
            <button
              type="button"
              class:active={activeCells.has(keyFor(row, col))}
              data-row={row}
              data-col={col}
              aria-label={`Cell ${row + 1}, ${col + 1}`}
            ></button>
          {/each}
        {/each}
      </div>
    </div>
  </section>

  <section class="panel export">
    <div class="export-header">
      <h2>SVG output</h2>
    </div>
    <textarea readonly rows="12" bind:value={svgOutput}></textarea>
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
    gap: 1.5rem;
    align-items: flex-start;
  }

  .controls-column {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  .input-row {
    display: flex;
    gap: 1rem;
    flex-wrap: wrap;
    align-items: flex-end;
  }

  label {
    display: flex;
    flex-direction: column;
    gap: 0.35rem;
    font-size: 0.9rem;
    color: #2b323b;
  }

  input[type="number"],
  input[type="text"] {
    border: 1px solid #d0d7de;
    border-radius: 0.6rem;
    padding: 0.5rem 0.7rem;
    font-size: 0.9rem;
  }

  input[type="color"] {
    width: 100%;
    border: 1px solid #d0d7de;
    border-radius: 0.6rem;
    padding: 0.3rem;
    height: 2.6rem;
  }

  .color {
    display: grid;
    gap: 0.5rem;
  }

  .color-row {
    display: flex;
    gap: 0.5rem;
    align-items: center;
  }

  .color-row input[type="color"] {
    flex: 0 0 3rem;
  }

  .color-row input[type="text"] {
    flex: 0 1 8rem;
    width: 8rem;
  }

  .actions {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
    align-items: flex-start;
    width: max-content;
  }

  .actions button {
    border: none;
    border-radius: 999px;
    padding: 0.55rem 1rem;
    background: #2b323b;
    color: white;
    font-size: 0.9rem;
    cursor: pointer;
    width: 100%;
  }

  .actions button:nth-of-type(2) {
    background: #4c6ef5;
  }

  .actions button:nth-of-type(3) {
    background: #12b886;
  }

  .status {
    font-size: 0.85rem;
    color: #16a34a;
  }

  .grid-panel {
    margin-top: 1.5rem;
    overflow: auto;
    padding: 1rem;
    border-radius: 1rem;
    border: 1px solid #e2e8f0;
    background: #f8fafc;
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

  .export-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
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
