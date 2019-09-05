<script>
  import { onMount, tick } from "svelte";
  export let id = "";

  // Styling
  export let primary = false;
  export let success = false;
  export let warning = false;
  export let danger = false;
  export let dark = false;

  let x1 = 100;
  let y1 = 0;
  let x2 = 100;
  let y2 = 100;
  let svgWidth = 100;
  let svgHeight = 100;

  const nodes = [["A", "Bverylong", "C"], ["D", "E", "F"]];
  let connectors = [
    { from: "A", to: "E", x1: 0, y1: 0, x2: 0, y2: 0 },
    { from: "C", to: "F", x1: 0, y1: 0, x2: 0, y2: 0 },
    { from: "F", to: "C", x1: 0, y1: 0, x2: 0, y2: 0 }
  ];

  let divs = {
    A: { ref: undefined },
    Bverylong: { ref: undefined },
    C: { ref: undefined },
    D: { ref: undefined },
    E: { ref: undefined },
    F: { ref: undefined }
  };

  function bbox(id) {
    const comp = divs[id].ref;
    if (comp && comp.offsetParent) {
      x1 = comp.offsetLeft + comp.offsetParent.offsetLeft;
      y1 = comp.offsetTop + comp.offsetParent.offsetTop;
      x2 = x1 + comp.offsetWidth;
      y2 = y1 + comp.offsetHeight;
      return { x1, y1, x2, y2 };
    } else {
      return undefined;
    }
  }

  function middle(a, b) {
    return (a + b) / 2;
  }

  onMount(async () => {
    console.log("onMount", divs.A.ref.offsetParent);
    setTimeout(() => {
      console.log("timeout", divs.A.ref.offsetParent);
      connectors = connectors.map(conn => {
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

  // $: console.log("connectors", connectors);

  /*
   If fixed is passed as an attribute, the close (X) icon does not 
   show up and the user cannot close the alert-box;
   */
  export let fixed = false;

  /*
  Create a custom "close" event that is fired when the user clicks on the close (X) icon.
  Users can subscribe to this event by targeting the custom element and adding an event
  listener for this custom event. It's completely up to the end user to decide how they want to 
  handle the closing of the element. i.e hidden vs. display, apply animation, etc...
  This is demonstrated in the index.html file.
  */
  function dispatchCloseEvent(e) {
    // 1. Create the custom event.
    const event = new CustomEvent("close", {
      detail: `alert-box was closed.`,
      bubbles: true,
      cancelable: true,
      composed: true // makes the event jump shadow DOM boundary
    });

    // 2. Dispatch the custom event.
    this.dispatchEvent(event);
  }
</script>

<style>
  /* 
  Setting custom css variables enables the user to use css to target a custom
  element by an attribute and change css properties that you want to expose.
  */
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

  :host([primary]) {
    --alert-box-bg: #cce5ff;
    --alert-box-text: #004085;
  }
  :host([success]) {
    --alert-box-bg: #d4edda;
    --alert-box-text: #155724;
  }
  :host([warning]) {
    --alert-box-bg: #fff3cd;
    --alert-box-text: #856404;
  }
  :host([danger]) {
    --alert-box-bg: #f8d7da;
    --alert-box-text: #721c24;
  }

  :host([dark]) {
    --alert-box-bg: #292b2c;
    --alert-box-text: #cccccc;
  }

  path.connector {
    stroke-width: 2px;
    stroke: rgb(0, 0, 0);
    fill: transparent;
  }

  svg {
    position: absolute;
    top: 0;
    left: 0;
  }

  table {
    border: 1px solid blue;
    background-color: lightblue;
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
    {#each connectors as conn}
      <path
        class="connector"
        d="M {conn.x1}
        {conn.y1} C {middle(conn.x1, conn.x2)}
        {conn.y1}
        {middle(conn.x1, conn.x2)}
        {conn.y2}
        {conn.x2}
        {conn.y2}"
        marker-end="url(#head)" />
    {/each}
  </svg>
  <table>
    <tbody>
      {#each nodes as row}
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
