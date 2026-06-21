<script lang="ts">
  let { fromX, fromY, toX, toY } = $props();
</script>

<svg>
  <path
    d="M {fromX},{fromY} C {fromX + 80},{fromY} {toX - 80},{toY} {toX},{toY}"
    fill="none"
    stroke="#22c55e"
    stroke-width="2"
    stroke-dasharray="5,5"
  />
</svg>