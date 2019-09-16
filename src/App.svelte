<script>
    import { afterUpdate, onDestroy, tick } from 'svelte'

    const internalPrefix = '__gg__'
    const nodeRadius = 5
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

    // _connectors is an array of of objects in the form
    // {
    //   from: string
    //   to: string
    //   color?: string
    //   x1?: number
    //   y1?: number
    //   x2?: number
    //   y2?: number
    // }
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

                const sideTop = node.box.y1 + nodeRadius
                const sideHeight = node.box.y2 - node.box.y1 - 2 * nodeRadius

                // Spread out all connectors from this node
                // to a node to the right on the right side of this node
                const connFromRight = _connectors
                    .filter(conn => conn.from === node.id)
                    .filter(conn => {
                        const toNode = nodeMap.get(conn.to)
                        return toNode && toNode.box.x1 > node.box.x2
                    })
                    .sort((a, b) => {
                        // Sort connectors based on the y-position
                        // of the node they are connecting to
                        const nodeA = nodeMap.get(a.to)
                        const nodeB = nodeMap.get(b.to)
                        return nodeA.box.y1 - nodeB.box.y1
                    })

                const sideSectionRight = sideHeight / (connFromRight.length + 1)

                connFromRight.forEach((conn, i) => {
                    conn.x1 = node.box.x2
                    conn.y1 = sideTop + sideSectionRight * (i + 1)
                })

                // Spread out all connectors to this node
                // from a node to the left on the left side of this node
                const connToLeft = _connectors
                    .filter(conn => conn.to === node.id)
                    .filter(conn => {
                        const fromNode = nodeMap.get(conn.from)
                        return fromNode && fromNode.box.x2 < node.box.x1
                    })
                    .sort((a, b) => {
                        // Sort connectors based on the y-position
                        // of the node they are connecting from
                        const nodeA = nodeMap.get(a.from)
                        const nodeB = nodeMap.get(b.from)
                        return nodeA.box.y1 - nodeB.box.y1
                    })

                const sideSectionLeft = sideHeight / (connToLeft.length + 1)

                connToLeft.forEach((conn, i) => {
                    conn.x2 = node.box.x1 - 4
                    conn.y2 = sideTop + sideSectionLeft * (i + 1)
                })
            }
        }

        svgWidth = width
        svgHeight = height

        _connectors = _connectors.map(conn => {
            const node1 = nodeMap.get(conn.from)
            const node2 = nodeMap.get(conn.to)
            if (!node1) {
                console.log('No such node ' + conn.from)
                return { ...conn, x1: undefined }
            }
            if (!node2) {
                console.log('No such node ' + conn.to)
                return { ...conn, x1: undefined }
            }
            const box1 = node1.box
            const box2 = node2.box
            if (box1 && box2) {
                if (box1.x2 < box2.x1) {
                    // Already placed
                    return conn
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

    let latestTouchedId = undefined
    function dispatchClickEvent(e) {
        // The click event does not fire on ios safari when
        // using the component from angularjs, but the touchstart
        // event fires.
        const nodeId = e.target.dataset.id
        if (e.type === 'click' && nodeId === latestTouchedId) {
            latestTouchedId = undefined
            return
        }

        if (e.type === 'touchstart') {
            latestTouchedId = nodeId
        }

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
                                    on:click={dispatchClickEvent}
                                    on:touchstart={dispatchClickEvent}>
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
