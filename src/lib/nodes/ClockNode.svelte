<script lang="ts">
  import { Handle, Position } from '@xyflow/svelte';

  let { id, data }: { id: string; data: any } = $props();
  let editing = $state(false);
  let editValue = $state('');
  let labelValue = $derived(data.label ?? 'CLK');

  function startEdit() { editValue = labelValue; editing = true; }
  function stopEdit() { editing = false; data.onLabelChange?.(editValue); }
</script>

<div class="node" class:on={data.value}>
  {#if editing}
    <input
      class="label-input nodrag"
      bind:value={editValue}
      onblur={stopEdit}
      onkeydown={(e) => e.key === 'Enter' && stopEdit()}
      autofocus
    />
  {:else}
    <p class="label" ondblclick={startEdit}>{labelValue}</p>
  {/if}

  <div class="clock-display">
    <span class="val">{data.value ? '1' : '0'}</span>
    <span class="freq">{data.frequency ?? 1}Hz</span>
  </div>

  <div class="controls nodrag">
    <button onclick={() => data.onSpeedDown?.(id)}>−</button>
    <button onclick={() => data.onToggleClock?.(id)}>
      {data.running ? '⏹' : '▶'}
    </button>
    <button onclick={() => data.onSpeedUp?.(id)}>+</button>
  </div>

  <Handle type="source" position={Position.Right} />
</div>

<style>
  .node {
    background: #0f172a;
    border: 2px solid #334155;
    border-radius: 8px;
    padding: 8px 12px;
    min-width: 90px;
    text-align: center;
    font-family: monospace;
    transition: border-color 0.2s, box-shadow 0.2s;
  }
  .node.on {
    border-color: #22c55e;
    box-shadow: 0 0 10px rgba(34,197,94,0.3);
  }
  .label {
    margin: 0 0 4px;
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
    margin: 0 auto 4px;
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
  .clock-display {
    display: flex;
    justify-content: center;
    align-items: baseline;
    gap: 6px;
    margin-bottom: 6px;
  }
  .val {
    font-size: 22px;
    font-weight: bold;
    color: #334155;
  }
  .on .val { color: #4ade80; }
  .freq {
    font-size: 10px;
    color: #64748b;
  }
  .controls {
    display: flex;
    justify-content: center;
    gap: 4px;
  }
  .controls button {
    background: #1e293b;
    border: 1px solid #334155;
    border-radius: 4px;
    color: #94a3b8;
    font-size: 12px;
    width: 24px;
    height: 24px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  .controls button:hover {
    border-color: #22c55e;
    color: #4ade80;
  }
</style>