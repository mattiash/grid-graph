<script>
  import { onMount, tick } from "svelte";

  let svgWidth = 100;
  let svgHeight = 100;

  export let nodes = "[]";
  export let connectors = "[]";

  let _nodes = [];
  $: _nodes = JSON.parse(nodes);

  let _connectors = [];
  $: console.log(connectors);
  $: _connectors = JSON.parse(connectors);

  let divs = {};
  $: {
    _nodes.forEach(row => {
      row.forEach(n => {
        if (!divs[n]) {
          divs[n] = {};
        }
      });
    });
  }

  function bbox(id) {
    const comp = divs[id].ref;
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
            return { x1, y1, x2, y2 };
          } else if (middle(box2.x1, box2.x2) < box1.x1) {
            console.log("from cannot be to the right of to");
          } else {
            // same column
            if (box1.y1 < box2.y2) {
              // from above to
              const x1 = middle(box1.x1, box1.x2) + 5;
              const x2 = middle(box2.x1, box2.x2) + 5;
              const y1 = box1.y2;
              const y2 = box2.y1 - 4;
              return { x1, y1, x2, y2 };
            } else {
              const x1 = middle(box1.x1, box1.x2) - 5;
              const x2 = middle(box2.x1, box2.x2) - 5;
              const y1 = box1.y1;
              const y2 = box2.y2 + 4;
              return { x1, y1, x2, y2 };
            }
          }
        }
      });
    }, 100);

    svgWidth = 500;
    svgHeight = 500;
  });
</script>

<style>
  .container {
    position: relative;
  }

  .node {
    border-radius: 5px;
    padding: 16px;
    /* Customizable Styles */
    background: var(--alert-box-bg, #e2e3e5);
    color: var(--alert-box-text, #383d41);
    margin: 30px;
    text-align: center;
    min-width: 50px;
  }

  svg {
    position: absolute;
    top: 0;
    left: 0;
  }

  path.connector {
    stroke-width: 2px;
    stroke: rgb(0, 0, 0);
    fill: transparent;
  }

  td {
    padding: 0;
  }
</style>

<!-- 
This tells the Svelte compiler that this file is a custom element. 
We also have to include the "customElement: true" compiler setting in rollup configuration.
-->
<svelte:options tag="alert-box" />
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
        <path d="M0,0 V6 L3,3 Z" fill="black" />
      </marker>
    </defs>
    {#each _connectors as conn}
      <path
        class="connector"
        d="M {conn.x1 || 0}
        {conn.y1 || 0} C {middle(conn.x1 || 0, conn.x2 || 0)}
        {conn.y1 || 0}
        {middle(conn.x1 || 0, conn.x2 || 0)}
        {conn.y2 || 0}
        {conn.x2 || 0}
        {conn.y2 || 0}"
        marker-end="url(#head)" />
    {/each}
  </svg>
  <table>
    <tbody>
      {#each _nodes as row}
        <tr>
          {#each row as node}
            <td>
              <div class="node" bind:this={divs[node].ref}>{node}</div>
            </td>
          {/each}
        </tr>
      {/each}
    </tbody>
  </table>
</div>
