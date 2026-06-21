<script lang="ts">
  import { Handle, Position } from '@xyflow/svelte';

  let { data }: { data: any } = $props();
  let editing = $state(false);
  let editValue = $state('');
  let labelValue = $derived(data.label ?? 'OUT');

  function startEdit() { editValue = labelValue; editing = true; }
  function stopEdit() { editing = false; data.onLabelChange?.(editValue); }
</script>

<div class="output" class:on={data.value}>
  <Handle type="target" position={Position.Left} />

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

  <span class="val">{data.value ? '1' : '0'}</span>
</div>

<style>
  .output {
    background: #0f172a;
    border: 2px solid #334155;
    border-radius: 50%;
    width: 76px;
    height: 76px;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 2px;
    font-family: monospace;
    transition: border-color 0.2s, box-shadow 0.2s;
  }
  .output.on {
    border-color: #22c55e;
    box-shadow: 0 0 18px rgba(34,197,94,0.45);
    background: #052e16;
  }
  .label {
    margin: 0;
    font-size: 10px;
    color: #64748b;
    letter-spacing: 0.15em;
    cursor: pointer;
    text-transform: uppercase;
  }
  .label:hover { color: #4ade80; }
  .label-input {
    width: 55px;
    background: #1e293b;
    border: 1px solid #22c55e;
    border-radius: 4px;
    color: #4ade80;
    font-family: monospace;
    font-size: 10px;
    text-align: center;
    padding: 2px 4px;
    outline: none;
  }
  .val {
    font-size: 26px;
    font-weight: bold;
    color: #1e293b;
  }
  .on .val { color: #4ade80; }
</style>