<script>
  import { onMount, tick } from "svelte";

  let svgWidth = 100;
  let svgHeight = 100;

  export let nodes = "[]";
  export let connectors = "[]";

  let _nodes = [];
  $: _nodes = JSON.parse(nodes);

  let _connectors = [];
  $: _connectors = JSON.parse(connectors);

  let divs = {};
  $: {
    _nodes.forEach(row => {
      row.forEach(n => {
        if (n && !divs[n.id]) {
          divs[n.id] = { ref: null };
        }
      });
    });
  }

  function bbox(id) {
    const comp = divs[id] && divs[id].ref;
    if (comp && comp.offsetParent) {
      const x1 = comp.offsetLeft + comp.offsetParent.offsetLeft;
      const y1 = comp.offsetTop + comp.offsetParent.offsetTop;
      const x2 = x1 + comp.offsetWidth;
      const y2 = y1 + comp.offsetHeight;
      return { x1, y1, x2, y2 };
    } else {
      return undefined;
    }
  }

  function middle(a, b) {
    return (a + b) / 2;
  }

  onMount(async () => {
    setTimeout(() => {
      let width = 0;
      let height = 0;
      _nodes.forEach(row => {
        row.forEach(node => {
          if (node) {
            const box = bbox(node.id);
            width = Math.max(box.x2, width);
            height = Math.max(box.y2, height);
          }
        });
      });
      svgWidth = width;
      svgHeight = height;

      _connectors = _connectors.map(conn => {
        const box1 = bbox(conn.from);
        const box2 = bbox(conn.to);
        if (box1 && box2) {
          if (box1.x2 < box2.x1) {
            // box1 left of box2
            const x1 = box1.x2;
            const x2 = box2.x1 - 4;
            const y1 = middle(box1.y1, box1.y2);
            const y2 = middle(box2.y1, box2.y2);
            return { ...conn, x1, y1, x2, y2 };
          } else if (middle(box2.x1, box2.x2) < box1.x1) {
            console.log("from cannot be to the right of to");
          } else {
            // same column
            if (box1.y1 < box2.y2) {
              // from above to
              const x1 = middle(box1.x1, box1.x2) + 5;
              const x2 = x1;
              const y1 = box1.y2;
              const y2 = box2.y1 - 4;
              return { ...conn, x1, y1, x2, y2 };
            } else {
              const x1 = middle(box1.x1, box1.x2) - 5;
              const x2 = x1;
              const y1 = box1.y1;
              const y2 = box2.y2 + 4;
              return { ...conn, x1, y1, x2, y2 };
            }
          }
        }
        return conn;
      });
    }, 100);
  });

  function dispatchClickEvent(e) {
    const nodeId = e.originalTarget.dataset.id;
    // 1. Create the custom event.
    const event = new CustomEvent("nodeclick", {
      detail: `grid-graph node click`,
      bubbles: true,
      cancelable: true,
      composed: true, // makes the event jump shadow DOM boundary
      detail: { nodeId }
    });
    // 2. Dispatch the custom event.
    this.dispatchEvent(event);
  }
</script>

<style>
  .container {
    position: relative;
  }

  .node {
    display: inline-block;
    border-radius: 5px;
    padding: 16px;
    margin: 30px;
    text-align: center;
    min-width: 50px;
    width: auto;
    z-index: 100;
    cursor: pointer;
  }

  svg {
    position: absolute;
    top: 0;
    left: 0;
    z-index: 0;
    pointer-events: none;
  }

  path {
    stroke-width: 2px;
    fill: transparent;
  }

  marker path {
    stroke-width: 0;
    fill: inherit;
  }

  td {
    padding: 0;
    text-align: center;
  }
</style>

<!-- 
This tells the Svelte compiler that this file is a custom element. 
We also have to include the "customElement: true" compiler setting in rollup configuration.
-->
<svelte:options tag="grid-graph" />
<div class="container">
  <svg style="width: {svgWidth}px; height: {svgHeight}px">
    <defs>
      <marker
        id="head"
        orient="auto"
        markerWidth="3"
        markerHeight="6"
        refX="0.1"
        refY="3">
        <path d="M0,0 V6 L3,3 Z" />
      </marker>
    </defs>
    {#each _connectors as conn (conn)}
      {#if conn.x1 !== undefined}
        <path
          d="M {conn.x1}
          {conn.y1} C {middle(conn.x1, conn.x2)}
          {conn.y1}
          {middle(conn.x1, conn.x2)}
          {conn.y2}
          {conn.x2}
          {conn.y2}"
          style="stroke: {conn.color || 'black'}"
          marker-end="url(#head)" />
      {/if}
    {/each}
  </svg>
  <table>
    <tbody>
      {#each _nodes as row}
        <tr>
          {#each row as node}
            <td>
              {#if node}
                <div
                  class="node"
                  style="color: {node.color || '#383d41'}; background: {node.background || ' #e2e3e5'}"
                  data-id={node.id}
                  bind:this={divs[node.id].ref}
                  on:click={dispatchClickEvent}>
                  {node.id}
                </div>
              {/if}
            </td>
          {/each}
        </tr>
      {/each}
    </tbody>
  </table>
</div>
