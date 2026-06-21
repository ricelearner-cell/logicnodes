<script lang="ts">
  import { Handle, Position } from '@xyflow/svelte';

  let { id, data }: { id: string; data: any } = $props();

  let editing   = $state(false);
  let editValue = $state('');
  let labelValue = $derived(data.label ?? 'SEG');

  function startEdit() { editValue = labelValue; editing = true; }
  function stopEdit()  { editing = false; data.onLabelChange?.(editValue); }

  // 4-bit BCD input: bits D3 D2 D1 D0 (D3 = MSB)
  let digit = $derived(() => {
    const d0 = !!data.d0;
    const d1 = !!data.d1;
    const d2 = !!data.d2;
    const d3 = !!data.d3;
    return (d3 ? 8 : 0) | (d2 ? 4 : 0) | (d1 ? 2 : 0) | (d0 ? 1 : 0);
  });

  //  Segment map for 0–F
  //  Segments:  a b c d e f g
  const SEGMENTS: Record<number, boolean[]> = {
  //             a      b      c      d      e      f      g
    0:  [true,  true,  true,  true,  true,  true,  false],
    1:  [false, true,  true,  false, false, false, false],
    2:  [true,  true,  false, true,  true,  false, true ],
    3:  [true,  true,  true,  true,  false, false, true ],
    4:  [false, true,  true,  false, false, true,  true ],
    5:  [true,  false, true,  true,  false, true,  true ],
    6:  [true,  false, true,  true,  true,  true,  true ],
    7:  [true,  true,  true,  false, false, false, false],
    8:  [true,  true,  true,  true,  true,  true,  true ],
    9:  [true,  true,  true,  true,  false, true,  true ],
    10: [true,  true,  true,  false, true,  true,  true ], // A
    11: [false, false, true,  true,  true,  true,  true ], // b
    12: [true,  false, false, true,  true,  true,  false], // C
    13: [false, true,  true,  true,  true,  false, true ], // d
    14: [true,  false, false, true,  true,  true,  true ], // E
    15: [true,  false, false, false, true,  true,  true ], // F
  };

  const HEX = '0123456789AbCdEF';

  let segs = $derived(SEGMENTS[digit()] ?? SEGMENTS[0]);
  let hexChar = $derived(HEX[digit()] ?? '0');
</script>

<div class="node">
  <!-- Label -->
  {#if editing}
    <input
      class="label-input nodrag"
      bind:value={editValue}
      onblur={stopEdit}
      onkeydown={(e) => e.key === 'Enter' && stopEdit()}
      autofocus
    />
  {:else}
    <!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
    <p class="label" ondblclick={startEdit}>{labelValue}</p>
  {/if}

  <!-- 7-segment display -->
  <div class="display">
    <svg viewBox="0 0 60 90" width="60" height="90" xmlns="http://www.w3.org/2000/svg">
      <!-- segment a — top -->
      <polygon
        points="8,4 52,4 48,10 12,10"
        class="seg" class:on={segs[0]}
      />
      <!-- segment b — top right -->
      <polygon
        points="52,4 56,8 54,44 48,40 50,8"
        class="seg" class:on={segs[1]}
      />
      <!-- segment c — bottom right -->
      <polygon
        points="54,46 56,50 54,86 50,82 48,50"
        class="seg" class:on={segs[2]}
      />
      <!-- segment d — bottom -->
      <polygon
        points="12,80 48,80 52,86 8,86"
        class="seg" class:on={segs[3]}
      />
      <!-- segment e — bottom left -->
      <polygon
        points="6,50 10,46 12,80 8,84 4,80"
        class="seg" class:on={segs[4]}
      />
      <!-- segment f — top left -->
      <polygon
        points="8,4 12,8 10,42 6,46 4,42"
        class="seg" class:on={segs[5]}
      />
      <!-- segment g — middle -->
      <polygon
        points="12,42 48,42 52,45 48,48 12,48 8,45"
        class="seg" class:on={segs[6]}
      />
    </svg>

    <!-- Hex readout -->
    <div class="hex-readout">{hexChar}</div>
  </div>

  <!-- 4 input handles with labels -->
  <div class="handle-labels">
    <span>D3</span>
    <span>D2</span>
    <span>D1</span>
    <span>D0</span>
  </div>

  <Handle type="target" position={Position.Left} id="d3" style="top: 62%" />
  <Handle type="target" position={Position.Left} id="d2" style="top: 71%" />
  <Handle type="target" position={Position.Left} id="d1" style="top: 80%" />
  <Handle type="target" position={Position.Left} id="d0" style="top: 89%" />
</div>

<style>
  .node {
    background: #0a1628;
    border: 2px solid #1e3a5f;
    border-radius: 10px;
    padding: 10px 14px 14px;
    min-width: 110px;
    text-align: center;
    font-family: monospace;
    position: relative;
    box-shadow: 0 0 20px rgba(34, 197, 94, 0.05);
    transition: box-shadow 0.3s;
  }

  .node:hover {
    box-shadow: 0 0 24px rgba(34, 197, 94, 0.15);
  }

  .label {
    margin: 0 0 8px;
    font-size: 11px;
    color: #64748b;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    cursor: pointer;
  }
  .label:hover { color: #4ade80; }

  .label-input {
    display: block;
    width: 70px;
    margin: 0 auto 8px;
    background: #1e293b;
    border: 1px solid #22c55e;
    border-radius: 4px;
    color: #4ade80;
    font-family: monospace;
    font-size: 11px;
    text-align: center;
    padding: 2px 4px;
    outline: none;
  }

  .display {
    display: flex;
    flex-direction: column;
    align-items: center;
    background: #050d1a;
    border-radius: 6px;
    padding: 8px 10px 6px;
    border: 1px solid #1e3a5f;
    margin-bottom: 8px;
  }

  /* OFF state — dim greenish */
  .seg {
    fill: #0d2a1a;
    transition: fill 0.06s, filter 0.06s;
  }

  /* ON state — bright green with glow */
  .seg.on {
    fill: #22c55e;
    filter: drop-shadow(0 0 4px #22c55e) drop-shadow(0 0 8px #16a34a);
  }

  .hex-readout {
    margin-top: 6px;
    font-size: 13px;
    font-family: monospace;
    color: #334155;
    letter-spacing: 0.15em;
  }

  .handle-labels {
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    gap: 2px;
    font-size: 9px;
    color: #334155;
    font-family: monospace;
    padding-left: 2px;
    margin-top: 2px;
  }

  .handle-labels span {
    height: 14px;
    line-height: 14px;
  }
</style>
