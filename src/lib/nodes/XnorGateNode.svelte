<script lang="ts">
  import { Handle, Position } from '@xyflow/svelte';

  
  
  let { data }: { data: any } = $props();
let editing = $state(false);
let editValue = $state('');  // ← this line is missing
let labelValue = $derived(data.label ?? 'XNOR');

function startEdit() { editValue = labelValue; editing = true; }
function stopEdit() { editing = false; data.onLabelChange?.(editValue); }
</script>

<div class="gate" class:on={data.value}>
<Handle 
  type="target" 
  position={Position.Left} 
  id="a" 
  style="top:30%; width:14px; height:14px; background:#1e293b; border: 2px solid #475569;" 
/>
<Handle 
  type="target" 
  position={Position.Left} 
  id="b" 
  style="top:70%; width:14px; height:14px; background:#1e293b; border: 2px solid #475569;" 
/>
{#if editing}
    <input
      class="label-input nodrag"
      bind:value={editValue}
      onblur={stopEdit}
      onkeydown={(e) => e.key === 'Enter' && stopEdit()}
      autofocus
    />
  {:else}
    <span class="label" ondblclick={startEdit}>{labelValue}</span>
  {/if}
  
  <span class="out">{data.value ? '1' : '0'}</span>
  <Handle 
  type="source" 
  position={Position.Right}
  style="width:14px; height:14px; background:#1e293b; border: 2px solid #475569;"
/>
</div>
<style>
  .gate {
    background: #0f172a;
    border: 2px solid #334155;
    border-radius: 6px 20px 20px 6px;
    padding: 12px 20px;
    min-width: 80px;
    text-align: center;
    font-family: monospace;
    transition: border-color 0.2s, box-shadow 0.2s;
  }
  .gate.on {
    border-color: #22c55e;
    box-shadow: 0 0 10px rgba(34,197,94,0.3);
  }
  .label {
    display: block;
    font-size: 13px;
    font-weight: bold;
    color: #94a3b8;
    letter-spacing: 0.1em;
    cursor: pointer;
  }
  .label:hover { color: #4ade80; }
  .on .label { color: #86efac; }
  .label-input {
    display: block;
    width: 70px;
    background: #1e293b;
    border: 1px solid #22c55e;
    border-radius: 4px;
    color: #4ade80;
    font-family: monospace;
    font-size: 12px;
    text-align: center;
    padding: 2px 4px;
    outline: none;
  }
  .out {
    display: block;
    font-size: 18px;
    font-weight: bold;
    color: #334155;
    margin-top: 4px;
  }
  .on .out { color: #4ade80; }
</style>