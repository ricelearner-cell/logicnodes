<script lang="ts">
  import { SvelteFlow, Background, Controls, Panel, MiniMap } from '@xyflow/svelte';
  import type { Node, Edge, Connection } from '@xyflow/svelte';
  import { onDestroy } from 'svelte';

  import '@xyflow/svelte/dist/style.css';

  import InputNode          from './lib/nodes/InputNode.svelte';
  import AndGateNode        from './lib/nodes/AndGateNode.svelte';
  import OutputNode         from './lib/nodes/OutputNode.svelte';
  import OrGateNode         from './lib/nodes/OrGateNode.svelte';
  import NotGateNode        from './lib/nodes/NotGateNode.svelte';
  import NandGateNode       from './lib/nodes/NandGateNode.svelte';
  import NorGateNode        from './lib/nodes/NorGateNode.svelte';
  import XorGateNode        from './lib/nodes/XorGateNode.svelte';
  import XnorGateNode       from './lib/nodes/XnorGateNode.svelte';
  import ConstantNode       from './lib/nodes/ConstantNode.svelte';
  import ClockNode          from './lib/nodes/ClockNode.svelte';
  import SevenSegmentNode   from './lib/nodes/SevenSegmentNode.svelte';

  const nodeTypes = {
    input:  InputNode,
    and:    AndGateNode,
    output: OutputNode,
    or:     OrGateNode,
    not:    NotGateNode,
    nand:   NandGateNode,
    nor:    NorGateNode,
    xor:    XorGateNode,
    xnor:   XnorGateNode,
    high:   ConstantNode,
    low:    ConstantNode,
    clock:  ClockNode,
    seg7:   SevenSegmentNode,
  };

  function evaluateGate(type: string, vals: boolean[]): boolean {
    switch (type) {
      case 'and':  return vals.length >= 2 ? vals.every(Boolean) : false;
      case 'or':   return vals.length >= 1 ? vals.some(Boolean)  : false;
      case 'not':  return vals.length >= 1 ? !vals[0]            : true;
      case 'nand': return vals.length >= 2 ? !vals.every(Boolean): true;
      case 'nor':  return vals.length >= 2 ? !vals.some(Boolean) : true;
      case 'xor':  return vals.length >= 2 ? vals.filter(Boolean).length === 1 : false;
      case 'xnor': return vals.length >= 2 ? vals.filter(Boolean).length !== 1 : true;
      default:     return false;
    }
  }

  function stripCallbacks(data: any) {
    const { onToggle, onLabelChange, onToggleClock, onSpeedUp, onSpeedDown, ...rest } = data;
    return rest;
  }

  let clockIntervals: Record<string, ReturnType<typeof setInterval>> = {};

  function startClock(id: string, freq: number) {
    clockIntervals[id] = setInterval(() => {
      nodes = nodes.map(n =>
        n.id === id ? { ...n, data: { ...n.data, value: !n.data.value } } : n
      );
      propagate(nodes, edges);
    }, 500 / freq);
    nodes = nodes.map(n =>
      n.id === id ? { ...n, data: { ...n.data, running: true } } : n
    );
  }

  function toggleClock(id: string) {
    const node = nodes.find(n => n.id === id);
    if (!node) return;
    if (clockIntervals[id]) {
      clearInterval(clockIntervals[id]);
      delete clockIntervals[id];
      nodes = nodes.map(n =>
        n.id === id ? { ...n, data: { ...n.data, running: false } } : n
      );
    } else {
      startClock(id, (node.data.frequency as number) ?? 1);
    }
  }

  function clockSpeedUp(id: string) {
    const node = nodes.find(n => n.id === id);
    if (!node) return;
    const freq = Math.min((node.data.frequency as number ?? 1) + 1, 16);
    nodes = nodes.map(n =>
      n.id === id ? { ...n, data: { ...n.data, frequency: freq } } : n
    );
    if (clockIntervals[id]) {
      clearInterval(clockIntervals[id]);
      delete clockIntervals[id];
      startClock(id, freq);
    }
  }

  function clockSpeedDown(id: string) {
    const node = nodes.find(n => n.id === id);
    if (!node) return;
    const freq = Math.max((node.data.frequency as number ?? 1) - 1, 1);
    nodes = nodes.map(n =>
      n.id === id ? { ...n, data: { ...n.data, frequency: freq } } : n
    );
    if (clockIntervals[id]) {
      clearInterval(clockIntervals[id]);
      delete clockIntervals[id];
      startClock(id, freq);
    }
  }

  function toggle(id: string) {
    snapshot();
    nodes = nodes.map(n =>
      n.id === id ? { ...n, data: { ...n.data, value: !n.data.value } } : n
    );
    propagate(nodes, edges);
  }

  function updateLabel(id: string, label: string) {
    nodes = nodes.map(n =>
      n.id === id ? { ...n, data: { ...n.data, label } } : n
    );
  }

  function makeCallbacks(id: string, type: string) {
    return {
      onLabelChange: (l: string) => updateLabel(id, l),
      ...(type === 'input' ? { onToggle: toggle } : {}),
      ...(type === 'clock' ? {
        onToggleClock: toggleClock,
        onSpeedUp:     clockSpeedUp,
        onSpeedDown:   clockSpeedDown,
      } : {}),
    };
  }

  let prevSig: Record<string, boolean> = {};

  function propagate(ns: Node[], es: Edge[]) {
    const sig: Record<string, boolean> = { ...prevSig };

    ns.forEach(n => {
      if (n.type === 'input') sig[n.id] = !!n.data.value;
      if (n.type === 'high')  sig[n.id] = true;
      if (n.type === 'low')   sig[n.id] = false;
      if (n.type === 'clock') sig[n.id] = !!n.data.value;
    });

    for (let pass = 0; pass < 20; pass++) {
      let changed = false;
      ns.forEach(n => {
        if (['input', 'high', 'low', 'clock', 'seg7'].includes(n.type ?? '')) return;
        const vals = es
          .filter(e => e.target === n.id)
          .map(e => sig[e.source] ?? false);
        const newVal = n.type === 'output'
          ? (vals[0] ?? false)
          : evaluateGate(n.type ?? '', vals);
        if (sig[n.id] !== newVal) { sig[n.id] = newVal; changed = true; }
      });
      if (!changed) break;
    }

    // Resolve seg7 bit inputs from named handles
    ns.forEach(n => {
      if (n.type !== 'seg7') return;
      ['d0', 'd1', 'd2', 'd3'].forEach(bit => {
        const edge = es.find(e => e.target === n.id && e.targetHandle === bit);
        sig[`${n.id}_${bit}`] = edge ? (sig[edge.source] ?? false) : false;
      });
    });

    prevSig = { ...sig };

    nodes = ns.map(n => {
      if (n.type === 'seg7') {
        return {
          ...n,
          data: {
            ...n.data,
            d0: sig[`${n.id}_d0`] ?? false,
            d1: sig[`${n.id}_d1`] ?? false,
            d2: sig[`${n.id}_d2`] ?? false,
            d3: sig[`${n.id}_d3`] ?? false,
          },
        };
      }
      return { ...n, data: { ...n.data, value: sig[n.id] ?? n.data.value } };
    });

    edges = es.map(e => ({
      ...e,
      animated: !!sig[e.source],
      style: sig[e.source]
        ? 'stroke:#22c55e;stroke-width:2.5px'
        : 'stroke:#334155;stroke-width:2px',
    }));
  }

  type TruthTableRow = Record<string, boolean>;
  let truthTable     = $state.raw<TruthTableRow[]>([]);
  let showTruthTable = $state(false);

  function generateTruthTable() {
    const inputNodes  = nodes.filter(n => n.type === 'input');
    const outputNodes = nodes.filter(n => n.type === 'output');
    const n = inputNodes.length;
    const rows: TruthTableRow[] = [];

    for (let i = 0; i < Math.pow(2, n); i++) {
      const testNodes = nodes.map(node => {
        const idx = inputNodes.findIndex(inp => inp.id === node.id);
        if (idx !== -1) {
          return { ...node, data: { ...node.data, value: !!(i & (1 << (n - 1 - idx))) } };
        }
        return node;
      });

      const sig: Record<string, boolean> = {};
      testNodes.forEach(node => {
        if (node.type === 'input') sig[node.id] = !!node.data.value;
        if (node.type === 'high')  sig[node.id] = true;
        if (node.type === 'low')   sig[node.id] = false;
      });

      for (let pass = 0; pass < 20; pass++) {
        let changed = false;
        testNodes.forEach(node => {
          if (['input', 'high', 'low', 'clock', 'seg7'].includes(node.type ?? '')) return;
          const vals = edges
            .filter(e => e.target === node.id)
            .map(e => sig[e.source] ?? false);
          const newVal = node.type === 'output'
            ? (vals[0] ?? false)
            : evaluateGate(node.type ?? '', vals);
          if (sig[node.id] !== newVal) { sig[node.id] = newVal; changed = true; }
        });
        if (!changed) break;
      }

      const row: TruthTableRow = {};
      inputNodes.forEach(inp  => { row[String(inp.data.label ?? inp.id)]  = sig[inp.id]; });
      outputNodes.forEach(out => { row[String(out.data.label ?? out.id)]  = sig[out.id] ?? false; });
      rows.push(row);
    }

    truthTable     = rows;
    showTruthTable = true;
  }

  let nodes = $state.raw<Node[]>([
    { id: 'a',   type: 'input',  position: { x: 60,  y: 100 }, data: { label: 'A', value: false, ...makeCallbacks('a',   'input')  } },
    { id: 'b',   type: 'input',  position: { x: 60,  y: 240 }, data: { label: 'B', value: false, ...makeCallbacks('b',   'input')  } },
    { id: 'and', type: 'and',    position: { x: 280, y: 155 }, data: { value: false,              ...makeCallbacks('and', 'and')    } },
    { id: 'out', type: 'output', position: { x: 500, y: 175 }, data: { value: false,              ...makeCallbacks('out', 'output') } },
  ]);

  let edges = $state.raw<Edge[]>([
    { id: 'a-and',   source: 'a',   target: 'and', targetHandle: 'a' },
    { id: 'b-and',   source: 'b',   target: 'and', targetHandle: 'b' },
    { id: 'and-out', source: 'and', target: 'out' },
  ]);

  type HistoryEntry = { nodes: Node[]; edges: Edge[] };
  let history: HistoryEntry[] = [];
  let historyIndex = -1;

  function snapshot() {
    history = history.slice(0, historyIndex + 1);
    history.push({
      nodes: nodes.map(n => ({ ...n, data: stripCallbacks(n.data) })),
      edges: [...edges],
    });
    historyIndex = history.length - 1;
  }
  snapshot();

  function undo() {
    if (historyIndex <= 0) return;
    historyIndex--;
    restore(history[historyIndex]);
  }

  function redo() {
    if (historyIndex >= history.length - 1) return;
    historyIndex++;
    restore(history[historyIndex]);
  }

  function restore(entry: HistoryEntry) {
    Object.keys(clockIntervals).forEach(id => {
      clearInterval(clockIntervals[id]);
      delete clockIntervals[id];
    });
    nodes = entry.nodes.map(n => ({
      ...n,
      data: {
        ...n.data,
        ...makeCallbacks(n.id, n.type ?? ''),
        ...(n.type === 'clock' ? { running: false } : {}),
      },
    }));
    edges = entry.edges;
    propagate(nodes, edges);
  }

  function handleKeydown(e: KeyboardEvent) {
    if ((e.ctrlKey || e.metaKey) && e.key === 'z' && !e.shiftKey) { e.preventDefault(); undo(); }
    if ((e.ctrlKey || e.metaKey) && (e.key === 'y' || (e.key === 'z' && e.shiftKey))) { e.preventDefault(); redo(); }
  }

  function saveCircuit() {
    const circuit = {
      nodes: nodes.map(n => ({ ...n, data: stripCallbacks(n.data) })),
      edges: edges.map(e => ({
        id: e.id, source: e.source, target: e.target,
        sourceHandle: e.sourceHandle, targetHandle: e.targetHandle,
      })),
    };
    const blob = new Blob([JSON.stringify(circuit, null, 2)], { type: 'application/json' });
    const url  = URL.createObjectURL(blob);
    const a    = document.createElement('a');
    a.href = url; a.download = 'circuit.json'; a.click();
    URL.revokeObjectURL(url);
  }

  function loadCircuit(e: Event) {
    const file = (e.target as HTMLInputElement).files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (ev) => {
      const circuit = JSON.parse(ev.target?.result as string);
      nodes = circuit.nodes.map((n: Node) => ({
        ...n,
        data: { ...n.data, ...makeCallbacks(n.id, n.type ?? '') },
      }));
      edges = circuit.edges;
      propagate(nodes, edges);
    };
    reader.readAsText(file);
  }

  // ── Share via URL ────────────────────────────────────────────────────
  let shareToast = $state('');

  function circuitToHash(): string {
    const circuit = {
      nodes: nodes.map(n => ({ ...n, data: stripCallbacks(n.data) })),
      edges: edges.map(e => ({
        id: e.id, source: e.source, target: e.target,
        sourceHandle: e.sourceHandle, targetHandle: e.targetHandle,
      })),
    };
    return btoa(unescape(encodeURIComponent(JSON.stringify(circuit))));
  }

  function loadFromHash(hash: string) {
    try {
      const circuit = JSON.parse(decodeURIComponent(escape(atob(hash))));
      nodes = circuit.nodes.map((n: Node) => ({
        ...n,
        data: { ...n.data, ...makeCallbacks(n.id, n.type ?? '') },
      }));
      edges = circuit.edges;
      propagate(nodes, edges);
    } catch {
      console.warn('Invalid circuit hash in URL');
    }
  }

  function shareCircuit() {
    const hash = circuitToHash();
    const url  = `${window.location.origin}${window.location.pathname}#${hash}`;
    window.history.replaceState(null, '', `#${hash}`);
    navigator.clipboard.writeText(url).then(() => {
      shareToast = '🔗 Link copied!';
      setTimeout(() => shareToast = '', 2500);
    }).catch(() => {
      shareToast = '🔗 URL updated!';
      setTimeout(() => shareToast = '', 2500);
    });
  }

  // Load from URL hash on startup
  if (typeof window !== 'undefined' && window.location.hash.length > 1) {
    loadFromHash(window.location.hash.slice(1));
  }

  function addNode(type: string, extraData = {}) {
    snapshot();
    const id = `${type}-${Date.now()}`;
    nodes = [...nodes, {
      id, type,
      position: { x: type === 'input' ? 60 : type === 'output' ? 500 : 280, y: 100 + nodes.length * 80 },
      data: { value: false, ...extraData, ...makeCallbacks(id, type) },
    }];
  }

  function addInput() {
    snapshot();
    const label = String.fromCharCode(65 + nodes.filter(n => n.type === 'input').length);
    const id    = `input-${Date.now()}`;
    nodes = [...nodes, {
      id, type: 'input',
      position: { x: 60, y: 100 + nodes.length * 80 },
      data: { label, value: false, ...makeCallbacks(id, 'input') },
    }];
  }

  function onconnect(_: Connection) {
    snapshot();
    setTimeout(() => propagate(nodes, edges), 50);
  }

  function ondelete(_: any) {
    snapshot();
    setTimeout(() => propagate(nodes, edges), 50);
  }

  onDestroy(() => {
    Object.values(clockIntervals).forEach(clearInterval);
  });
</script>

<!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
<!-- svelte-ignore a11y_no_noninteractive_tabindex -->
<div class="canvas" role="region" aria-label="Logic Gate Canvas" onkeydown={handleKeydown} tabindex="0">
  <SvelteFlow
    bind:nodes
    bind:edges
    {nodeTypes}
    {onconnect}
    {ondelete}
    snapGrid={[20, 20]}
    fitView
  >
    <Background gap={24} />
    <Controls />
    <MiniMap nodeColor="#1e293b" maskColor="rgba(15, 23, 42, 0.8)" />

    <Panel position="top-right">
      <button onclick={saveCircuit} class="add-btn">💾 Save</button>
      <label class="add-btn" style="cursor:pointer">
        📂 Load
        <input type="file" accept=".json" onchange={loadCircuit} style="display:none" />
      </label>
      <hr style="border-color:#334155; margin: 4px 0" />
      <button onclick={() => addNode('and',  { value: false })} class="add-btn">+ AND Gate</button>
      <button onclick={() => addNode('or',   { value: false })} class="add-btn">+ OR Gate</button>
      <button onclick={() => addNode('not',  { value: true  })} class="add-btn">+ NOT Gate</button>
      <button onclick={() => addNode('nand', { value: true  })} class="add-btn">+ NAND Gate</button>
      <button onclick={() => addNode('nor',  { value: true  })} class="add-btn">+ NOR Gate</button>
      <button onclick={() => addNode('xor',  { value: false })} class="add-btn">+ XOR Gate</button>
      <button onclick={() => addNode('xnor', { value: true  })} class="add-btn">+ XNOR Gate</button>
      <hr style="border-color:#334155; margin: 4px 0" />
      <button onclick={addInput}                                                                 class="add-btn">+ Input</button>
      <button onclick={() => addNode('output', { value: false })}                               class="add-btn">+ Output</button>
      <button onclick={() => addNode('high',   { value: true  })}                               class="add-btn">⬆ HIGH</button>
      <button onclick={() => addNode('low',    { value: false })}                               class="add-btn">⬇ LOW</button>
      <button onclick={() => addNode('clock',  { value: false, frequency: 1, running: false })} class="add-btn">⏱ Clock</button>
      <button onclick={() => addNode('seg7',   { d0: false, d1: false, d2: false, d3: false })} class="add-btn">🔢 7-Seg</button>
      <hr style="border-color:#334155; margin: 4px 0" />
      <button onclick={generateTruthTable} class="add-btn">📊 Truth Table</button>
      <button onclick={shareCircuit}        class="add-btn">🔗 Share</button>
      <button onclick={undo}               class="add-btn">↩ Undo</button>
      <button onclick={redo}               class="add-btn">↪ Redo</button>
    </Panel>
  </SvelteFlow>

  {#if showTruthTable && truthTable.length > 0}
    <div class="truth-table">
      <div class="tt-header">
        <span>Truth Table</span>
        <button onclick={() => showTruthTable = false} class="tt-close">✕</button>
      </div>
      <table>
        <thead>
          <tr>{#each Object.keys(truthTable[0]) as col}<th>{col}</th>{/each}</tr>
        </thead>
        <tbody>
          {#each truthTable as row}
            <tr>{#each Object.values(row) as val}<td class:one={val}>{val ? '1' : '0'}</td>{/each}</tr>
          {/each}
        </tbody>
      </table>
    </div>
  {/if}

  <p class="hint">Click inputs to toggle · Drag handles to connect · Backspace to delete</p>

  {#if shareToast}
    <div class="toast">{shareToast}</div>
  {/if}

</div>

<style>
  :global(body), :global(html) { margin: 0; padding: 0; overflow: hidden; }

  .canvas {
    height: 100vh;
    width: 100vw;
    position: relative;
    background: #0f172a;
    overflow: hidden;
  }

  :global(.svelte-flow__handle) {
    width: 16px !important;
    height: 16px !important;
    background: #1e293b !important;
    border: 2px solid #475569 !important;
  }
  :global(.svelte-flow__handle:hover) {
    background: #052e16 !important;
    border-color: #22c55e !important;
  }
  :global(.svelte-flow__panel) {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .add-btn {
    background: #1e293b;
    border: 1px solid #334155;
    border-radius: 6px;
    color: #94a3b8;
    font-family: monospace;
    font-size: 13px;
    padding: 6px 14px;
    cursor: pointer;
    transition: all 0.15s;
  }
  .add-btn:hover { border-color: #22c55e; color: #4ade80; }

  .hint {
    position: absolute;
    bottom: 16px;
    left: 50%;
    transform: translateX(-50%);
    margin: 0;
    padding: 5px 14px;
    background: #1e293b;
    border: 1px solid #334155;
    border-radius: 20px;
    color: #475569;
    font-size: 12px;
    font-family: monospace;
    pointer-events: none;
  }

  .truth-table {
    position: absolute;
    bottom: 60px;
    left: 50%;
    transform: translateX(-50%);
    background: #1e293b;
    border: 1px solid #334155;
    border-radius: 8px;
    padding: 12px;
    font-family: monospace;
    font-size: 13px;
    z-index: 10;
  }
  .tt-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    color: #94a3b8;
    margin-bottom: 8px;
    font-size: 12px;
    letter-spacing: 0.1em;
    text-transform: uppercase;
  }
  .tt-close { background: none; border: none; color: #475569; cursor: pointer; font-size: 14px; }
  .tt-close:hover { color: #4ade80; }
  table { border-collapse: collapse; }
  th { color: #64748b; padding: 4px 16px; border-bottom: 1px solid #334155; text-align: center; }
  td { color: #475569; padding: 4px 16px; text-align: center; }
  td.one { color: #4ade80; }
  tr:hover td { background: #0f172a; }

  .toast {
    position: absolute;
    top: 24px;
    left: 50%;
    transform: translateX(-50%);
    background: #1e293b;
    border: 1px solid #22c55e;
    border-radius: 20px;
    color: #4ade80;
    font-family: monospace;
    font-size: 13px;
    padding: 6px 18px;
    pointer-events: none;
    animation: fadein 0.2s ease;
    z-index: 20;
  }
  @keyframes fadein {
    from { opacity: 0; transform: translateX(-50%) translateY(-6px); }
    to   { opacity: 1; transform: translateX(-50%) translateY(0); }
  }
</style>