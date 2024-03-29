# grid-graph

![Build master](https://github.com/mattiash/grid-graph/workflows/Build%20master/badge.svg)
![Publish to npm](https://github.com/mattiash/grid-graph/workflows/Publish%20to%20npm/badge.svg)
![npm (scoped)](https://img.shields.io/npm/v/@mattiash/grid-graph)

Built from component template at [https://github.com/gojutin/svelte-custom-element](https://github.com/gojutin/svelte-custom-element).

## Usage

grid-graph is a [custom element](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements) that should be possible to use in any web framework and also in plain html.

### Example

```
<grid-graph nodes="..." connectors='...'/>
```

`nodes` should be a json-encoded string containing a two-dimensional array of nodes.
Each node is an object with the following properties:

-   id - an identifier for the node. Used as the label of the node and when defining connectors.
-   color - optional text-color for the node. Default #383d41
-   background - optional background for the node. Default #e2e3e5.
-   border - optional border for the node. Default none.

```javascript
;[[{ id: 'A' }, { id: 'B' }], [undefined, { id: 'C' }]]
```

`connectors` shall be an array of connector objects. Each connector object has the following properties:

-   from - the id of a node
-   to - the id of a node
-   color - the color of the connector. Default: #444.

```javascript
;[{ from: 'A', to: 'B' }, { from: 'A', to: 'C' }]
```

The grid-graph element emits a `nodeclick` whenever the user clicks a node,
with the id of the node in event.detail.nodeId.

See `public/index.html` for a complete example.

See [@mattiash/grid-graph-placement](https://github.com/mattiash/grid-graph-placement) for a module
that automatically places out your nodes on a grid.

## Styling

The grid-graph can be styled with css variables.

```html
<grid-graph style="--node-margin: 5px 30px; --node-padding: 2px 10px">
</grid-graph>
```

## Development

```
npm run dev
```

Open [http://localhost:5000](http://localhost:5000)

## Production

```
npm run build
```
