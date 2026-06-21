<script lang="ts">
  import { Handle, Position } from '@xyflow/svelte';

  let { id, data }: { id: string; data: any } = $props();
  let editing = $state(false);
  let editValue = $state('');

  let labelValue = $derived(data.label ?? 'A');

  function startEdit() {
    editValue = labelValue;
    editing = true;
  }
  function stopEdit() {
    editing = false;
    data.onLabelChange?.(editValue);
  }
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

  <button onclick={() => data.onToggle(id)}>
    {data.value ? '1' : '0'}
  </button>
  <Handle type="source" position={Position.Right} />
</div>

<style>
  .node {
    background: #0f172a;
    border: 2px solid #334155;
    border-radius: 8px;
    padding: 8px 12px;
    min-width: 72px;
    text-align: center;
    font-family: monospace;
    transition: border-color 0.2s, box-shadow 0.2s;
  }
  .node.on {
    border-color: #22c55e;
    box-shadow: 0 0 10px rgba(34,197,94,0.3);
  }
  .label {
    margin: 0 0 6px;
    font-size: 11px;
    color: #64748b;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    cursor: pointer;
  }
  .label:hover { color: #4ade80; }
  .label-input {
    display: block;
    width: 60px;
    margin: 0 auto 6px;
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
  button {
    width: 40px;
    height: 40px;
    font-size: 20px;
    font-weight: bold;
    font-family: monospace;
    background: #1e293b;
    border: 2px solid #334155;
    border-radius: 6px;
    color: #475569;
    cursor: pointer;
    transition: all 0.15s;
  }
  .on button {
    background: #052e16;
    border-color: #22c55e;
    color: #4ade80;
  }
</style>