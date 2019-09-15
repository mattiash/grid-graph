<script>
    import { afterUpdate, onDestroy, tick } from 'svelte'

    const internalPrefix = '__gg__'
    let svgWidth = 100
    let svgHeight = 100

    export let nodes = '[]'
    export let connectors = '[]'

    let placed = false
    let destroyed = false
    let nodeMap = new Map()

    // _nodes is a two-dimensional array with nodes.
    // Each node looks like this:
    // {
    //   id: string
    //   title?: string
    //   color?: string
    //   background?: string
    //   box?: {
    //     x1: number
    //     y1: number
    //     x2: number
    //     y2: number
    //   }
    // }
    let _nodes = []
    $: {
        _nodes = JSON.parse(nodes).map((row, y) => {
            return row.map((node, x) =>
                !!node ? node : { id: `${internalPrefix}${x}_${y}` },
            )
        })

        placed = false
    }

    let _connectors = []
    $: {
        _connectors = JSON.parse(connectors)
        placed = false
    }

    let divs = {}
    $: {
        nodeMap = new Map()

        _nodes.forEach(row => {
            row.forEach(node => {
                nodeMap.set(node.id, node)
                if (node && !divs[node.id]) {
                    divs[node.id] = { ref: null }
                }
            })
        })
    }

    function bbox(id) {
        const comp = divs[id] && divs[id].ref
        if (comp && comp.offsetParent) {
            const x1 = comp.offsetLeft + comp.offsetParent.offsetLeft
            const y1 = comp.offsetTop + comp.offsetParent.offsetTop
            const x2 = x1 + comp.offsetWidth
            const y2 = y1 + comp.offsetHeight
            return { x1, y1, x2, y2 }
        } else {
            return undefined
        }
    }

    function middle(a, b) {
        return (a + b) / 2
    }

    function addNodePositions() {
        for (let row of _nodes) {
            for (let node of row) {
                if (node && !node.id.startsWith(internalPrefix)) {
                    node.box = bbox(node.id)

                    if (!node.box) {
                        // The div has not been placed yet.
                        // Abort!
                        return false
                    }
                }
            }
        }
        return true
    }

    function placeArrows() {
        let width = 0
        let height = 0

        if (!addNodePositions()) {
            return false
        }

        for (const node of nodeMap.values()) {
            if (node.box) {
                width = Math.max(node.box.x2, width)
                height = Math.max(node.box.y2, height)
            }
        }

        svgWidth = width
        svgHeight = height

        _connectors = _connectors.map(conn => {
            const box1 = nodeMap.get(conn.from).box
            const box2 = nodeMap.get(conn.to).box
            if (box1 && box2) {
                if (box1.x2 < box2.x1) {
                    // box1 left of box2
                    const x1 = box1.x2
                    const x2 = box2.x1 - 4
                    const y1 = middle(box1.y1, box1.y2)
                    const y2 = middle(box2.y1, box2.y2)
                    return { ...conn, x1, y1, x2, y2 }
                } else if (middle(box2.x1, box2.x2) < box1.x1) {
                    console.log('from cannot be to the right of to')
                } else {
                    // same column
                    if (box1.y1 < box2.y2) {
                        // from above to
                        const x1 = middle(box1.x1, box1.x2) + 5
                        const x2 = x1
                        const y1 = box1.y2
                        const y2 = box2.y1 - 4
                        return { ...conn, x1, y1, x2, y2 }
                    } else {
                        const x1 = middle(box1.x1, box1.x2) - 5
                        const x2 = x1
                        const y1 = box1.y1
                        const y2 = box2.y2 + 4
                        return { ...conn, x1, y1, x2, y2 }
                    }
                }
            }
            return conn
        })

        return true
    }

    afterUpdate(async () => {
        while (!placed && !destroyed) {
            placed = placeArrows()
            await sleep(1000)
        }
    })

    onDestroy(() => {
        destroyed = true
    })

    function dispatchClickEvent(e) {
        const nodeId = e.originalTarget.dataset.id

        const event = new CustomEvent('nodeclick', {
            bubbles: true,
            cancelable: true,
            composed: true, // makes the event jump shadow DOM boundary
            detail: { nodeId },
        })

        this.dispatchEvent(event)
    }

    function sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms))
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
        margin: 10px 30px;
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
        {#each _connectors as conn (conn)}
            {#if conn.x1 !== undefined}
                <defs>
                    <marker
                        id={conn.from + conn.to}
                        orient="auto"
                        markerWidth="3"
                        markerHeight="6"
                        refX="0.1"
                        refY="3">
                        <path
                            d="M0,0 V6 L3,3 Z"
                            style="fill: {conn.color || 'black'}" />
                    </marker>
                </defs>
                {#if conn.x1 === conn.x2 && conn.y1 < conn.y2}
                    <!-- Straight down -->
                    <path
                        d="M {conn.x1}
                        {conn.y1} C {conn.x1 + 5}
                        {middle(conn.y1, conn.y2)}
                        {conn.x1 + 5}
                        {middle(conn.y1, conn.y2)}
                        {conn.x2}
                        {conn.y2}"
                        style="stroke: {conn.color || 'black'}"
                        marker-end="url(#{conn.from + conn.to})" />
                {:else if conn.x1 === conn.x2 && conn.y1 > conn.y2}
                    <!-- Straight up -->
                    <path
                        d="M {conn.x1}
                        {conn.y1} C {conn.x1 - 5}
                        {middle(conn.y1, conn.y2)}
                        {conn.x1 - 5}
                        {middle(conn.y1, conn.y2)}
                        {conn.x2}
                        {conn.y2}"
                        style="stroke: {conn.color || 'black'}"
                        marker-end="url(#{conn.from + conn.to})" />
                {:else}
                    <path
                        d="M {conn.x1}
                        {conn.y1} C {middle(conn.x1, conn.x2)}
                        {conn.y1}
                        {middle(conn.x1, conn.x2)}
                        {conn.y2}
                        {conn.x2}
                        {conn.y2}"
                        style="stroke: {conn.color || 'black'}"
                        marker-end="url(#{conn.from + conn.to})" />
                {/if}
            {/if}
        {/each}
    </svg>
    <table>
        <tbody>
            {#each _nodes as row (row)}
                <tr>
                    {#each row as node (node.id)}
                        <td>
                            {#if !node.id.startsWith(internalPrefix)}
                                <div
                                    class="node"
                                    style="color: {node.color || '#383d41'};
                                    background: {node.background || ' #e2e3e5'}"
                                    data-id={node.id}
                                    bind:this={divs[node.id].ref}
                                    on:click={dispatchClickEvent}>
                                    {node.title || node.id}
                                </div>
                            {/if}
                        </td>
                    {/each}
                </tr>
            {/each}
        </tbody>
    </table>
</div>
