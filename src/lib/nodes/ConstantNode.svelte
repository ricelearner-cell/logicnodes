<script lang="ts">
  import { Handle, Position } from '@xyflow/svelte';

  let { data }: { data: any } = $props();
</script>

<div class="node" class:high={data.value}>
  <span class="label">{data.value ? 'HIGH' : 'LOW'}</span>
  <span class="val">{data.value ? '1' : '0'}</span>
  <Handle type="source" position={Position.Right} />
</div>

<style>
  .node {
    background: #0f172a;
    border: 2px solid #334155;
    border-radius: 8px;
    padding: 8px 16px;
    text-align: center;
    font-family: monospace;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 4px;
    min-width: 72px;
    transition: border-color 0.2s, box-shadow 0.2s;
  }
  .node.high {
    border-color: #22c55e;
    box-shadow: 0 0 10px rgba(34,197,94,0.3);
  }
  .label {
    font-size: 10px;
    color: #64748b;
    letter-spacing: 0.1em;
    text-transform: uppercase;
  }
  .val {
    font-size: 20px;
    font-weight: bold;
    color: #334155;
  }
  .high .val { color: #4ade80; }
</style>