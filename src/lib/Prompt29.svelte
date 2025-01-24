<script lang="ts">
	import SVGContainer from '$lib/SVGContainer.svelte';

	const { svgId } = $props();
	let DEBUG = $state(false);

	const WIDTH = 1080;
	const HEIGHT = 720;
	const TILE_COUNT_X = 12;
	const TILE_SIZE_X = WIDTH / TILE_COUNT_X;
	const TILE_COUNT_Y = Math.round(HEIGHT / TILE_SIZE_X);
	const TILE_SIZE_Y = HEIGHT / TILE_COUNT_Y;

	type Tile = {
		x: number;
		y: number;
		width: number;
		height: number;
		index: number;
	}

	let tiles: Tile[] = Array.from({ length: TILE_COUNT_X * TILE_COUNT_Y }, (_, index) => {
		const x = (index % TILE_COUNT_X) * TILE_SIZE_X;
		const y = (Math.floor(index / TILE_COUNT_X)) * TILE_SIZE_Y;
		return { x, y, index, width: TILE_SIZE_X, height: TILE_SIZE_Y };
	});

	type Shape = {
		tiles: Tile[];
		d: string;
		color: string;
		index: number;
	};

	type Point = { x: number; y: number };
	type Edge = { start: Point; end: Point };

	function generatePathFromTiles(tiles: Tile[]): string {
		if (tiles.length === 0) return '';

		// Generate all edges for each tile
		const edges: Edge[] = [];
		tiles.forEach(tile => {
			// For each tile, create 4 edges
			edges.push(
				{ start: { x: tile.x, y: tile.y }, end: { x: tile.x + tile.width, y: tile.y } },             // top
				{ start: { x: tile.x + tile.width, y: tile.y }, end: { x: tile.x + tile.width, y: tile.y + tile.height } },  // right
				{ start: { x: tile.x + tile.width, y: tile.y + tile.height }, end: { x: tile.x, y: tile.y + tile.height } }, // bottom
				{ start: { x: tile.x, y: tile.y + tile.height }, end: { x: tile.x, y: tile.y } }            // left
			);
		});

		// Remove internal edges (edges that appear twice)
		const uniqueEdges: Edge[] = [];
		edges.forEach(edge => {
			const isDuplicate = edges.filter(e =>
				(arePointsEqual(e.start, edge.start) && arePointsEqual(e.end, edge.end)) ||
				(arePointsEqual(e.start, edge.end) && arePointsEqual(e.end, edge.start))
			).length > 1;

			if (!isDuplicate) {
				uniqueEdges.push(edge);
			}
		});

		// Sort edges to create a continuous path
		const path: Point[] = [];
		let currentEdge = uniqueEdges[0];
		let currentPoint = currentEdge.start;
		path.push(currentPoint);

		while (uniqueEdges.length > 0) {
			// Add the end point of current edge
			path.push(currentEdge.end);

			// Remove the current edge from uniqueEdges
			uniqueEdges.splice(uniqueEdges.indexOf(currentEdge), 1);

			// Find the next edge that connects to current end point
			currentPoint = currentEdge.end;
			currentEdge = uniqueEdges.find(edge =>
				arePointsEqual(edge.start, currentPoint) ||
				arePointsEqual(edge.end, currentPoint)
			)!;

			// If the edge is in wrong direction, flip it
			if (currentEdge && currentEdge.end && arePointsEqual(currentEdge.end, currentPoint)) {
				const temp = currentEdge.start;
				currentEdge.start = currentEdge.end;
				currentEdge.end = temp;
			}
			if (!currentEdge) break;
		}

		// Convert points to SVG path string
		return `M ${path[0].x} ${path[0].y} ` +
			path.slice(1).map(p => `L ${p.x} ${p.y}`).join(' ') +
			' Z';
	}

	function arePointsEqual(p1: Point, p2: Point): boolean {
		return Math.abs(p1.x - p2.x) < 0.001 && Math.abs(p1.y - p2.y) < 0.001;
	}

	const rotation = Math.random() * 137.508;
	let shapes: Shape[] = $state([{ tiles, d: generatePathFromTiles(tiles), color: getColor(0), index: 0 }]);

	// generate a color based on the index of the shape


	function getColor(index: number): string {
		const hue = (index * rotation) % 360;
		return `hsl(${hue}, 100%, 50%)`;
	}

	function subDivide() {
		let selectedShape = shapes.reduce((max, shape) => shape.tiles.length > max.tiles.length ? shape : max, shapes[0]);
		if (selectedShape.tiles.length <= 3) {
			console.warn('Cannot subdivide a shape with less than 2 tiles');
			return;
		}
		console.info('Selected shape', selectedShape.tiles.length);

		// calculate the width and height in tiles of the selected shape
		const minX = Math.min(...selectedShape.tiles.map(t => t.x));
		const maxX = Math.max(...selectedShape.tiles.map(t => t.x + t.width));
		const minY = Math.min(...selectedShape.tiles.map(t => t.y));
		const maxY = Math.max(...selectedShape.tiles.map(t => t.y + t.height));
		const widthInTiles = Math.ceil((maxX - minX) / TILE_SIZE_X);
		const heightInTiles = Math.ceil((maxY - minY) / TILE_SIZE_Y);
		const ratio = widthInTiles / heightInTiles;
		console.log('ratio', ratio, widthInTiles, heightInTiles);
		let horizontal = ratio < 1;
		if (ratio === 1) {
			horizontal = Math.random() > 0.5;
		}
		let newShapes: Shape[] = shapes.filter(s => s !== selectedShape);
		if (horizontal) {
			// split horizontally
			let splitY = (minY + TILE_SIZE_Y) + Math.floor(Math.random() * (heightInTiles - 2)) * TILE_SIZE_Y;
			let topTiles = selectedShape.tiles.filter(t => t.y <= splitY);
			let bottomTiles = selectedShape.tiles.filter(t => t.y > splitY);
			while (topTiles.length === 0 || bottomTiles.length === 0) {
				splitY -= 1;
				topTiles = selectedShape.tiles.filter(t => t.y <= splitY);
				bottomTiles = selectedShape.tiles.filter(t => t.y > splitY);
			}
			console.log('split hor', splitY, topTiles, bottomTiles);
			newShapes.push({ tiles: topTiles, d: generatePathFromTiles(topTiles), color: selectedShape.color, index: selectedShape.index });
			newShapes.push({ tiles: bottomTiles, d: generatePathFromTiles(bottomTiles), color: getColor(shapes.length), index: shapes.length });
		} else {
			// split vertically
			let splitX = (minX + TILE_SIZE_X) + Math.floor(Math.random() * (widthInTiles - 2)) * TILE_SIZE_X;
			let leftTiles = selectedShape.tiles.filter(t => t.x <= splitX);
			let rightTiles = selectedShape.tiles.filter(t => t.x > splitX);
			while (leftTiles.length === 0 || rightTiles.length === 0) {
				splitX -= 1;
				leftTiles = selectedShape.tiles.filter(t => t.x <= splitX);
				rightTiles = selectedShape.tiles.filter(t => t.x > splitX);
			}
			console.log('split ver', splitX, leftTiles, rightTiles);
			newShapes = shapes.filter(s => s !== selectedShape);
			newShapes.push({ tiles: leftTiles, d: generatePathFromTiles(leftTiles), color: selectedShape.color, index: selectedShape.index });
			newShapes.push({ tiles: rightTiles, d: generatePathFromTiles(rightTiles), color: getColor(shapes.length), index: shapes.length });
		}

		if (newShapes.filter(s => s.tiles.length === 0).length > 0) {
			// TODO when our shapes become too small (+- 2x2), the splitting fails
			console.error('Empty shapes found', newShapes);
			return;
		}
		shapes = newShapes;
	}

	function isInsideOfShape(point: Point, shape: Shape): boolean {
		for (let i = 0; i < shape.tiles.length; i++) {
			const tile = shape.tiles[i];
			if (point.x >= tile.x && point.x <= tile.x + tile.width && point.y >= tile.y && point.y <= tile.y + tile.height) {
				return true;
			}
		}
		return false;
	}

	function collectInternalPoints(shape: Shape): Point[] {
		const minX = Math.min(...shape.tiles.map(t => t.x));
		const maxX = Math.max(...shape.tiles.map(t => t.x + t.width));
		const minY = Math.min(...shape.tiles.map(t => t.y));
		const maxY = Math.max(...shape.tiles.map(t => t.y + t.height));
		const internalPoints: Point[] = [];
		for (let x = minX; x <= maxX; x += TILE_SIZE_X) {
			for (let y = minY; y <= maxY; y += TILE_SIZE_Y) {
				if (x <= 0 || x >= WIDTH || y <= 0 || y >= HEIGHT) {
					// canvas boundaries
					continue;
				}
				if (!isInsideOfShape({ x, y }, shape)) {
					// not inside the shape
					console.log('not inside', x, y);
					continue;
				}
				internalPoints.push({ x, y });
			}
		}
		return internalPoints;
	}

	function swap() {
		let newShapes = shapes.filter(s => true);
		// let selectedShape = newShapes.reduce((max, shape) => shape.tiles.length > max.tiles.length ? shape : max, newShapes[0]);
		let selectedShape = newShapes[Math.floor(Math.random() * newShapes.length)];
		if (selectedShape.tiles.length < 2) {
			console.warn('Cannot subdivide a shape with less than 2 tiles');
			return;
		}
		let points = collectInternalPoints(selectedShape);
		console.log('selectedShape', $state.snapshot(selectedShape), 'internal points', [...points]);
		// find the neighboring shape with the most common internal points

		const overlappingPointsMap = new Map<Shape, Point[]>();
		newShapes.forEach(s => {
			if (s.index === selectedShape.index) return;
			let currentPoints = collectInternalPoints(s);
			const overlappingPoints = currentPoints.filter(p => points.some(p2 => arePointsEqual(p, p2)));
			overlappingPointsMap.set(s, overlappingPoints);
		});
		// select shape with most overlap points as neighborShape
		let neighborShape = newShapes.reduce((max, shape) => {
			const points = overlappingPointsMap.get(shape);
			return points && points.length > (overlappingPointsMap.get(max)?.length || 0) ? shape : max;
		}, newShapes[0]);
		const ols = overlappingPointsMap.get(neighborShape);
		console.log('neighborShape', $state.snapshot(neighborShape), 'overlapping points', ols ? [...ols] : '--');

		if (!selectedShape || !neighborShape) {
			console.warn('Could not find two neighboring shapes to swap');
			return;
		}
		// collect all common internal points for both shapes
		let selectedPoints = collectInternalPoints(selectedShape);
		let neighborPoints = collectInternalPoints(neighborShape);
		let commonPoints = selectedPoints.filter(p => neighborPoints.some(p2 => arePointsEqual(p, p2)));
		console.log('commonPoints', commonPoints);

		const selectedPoint = commonPoints[Math.floor(Math.random() * commonPoints.length)];
		// get the neighboring tile that has the selected point
		const selectedTile = selectedShape.tiles.find(t => {
			return t.x <= selectedPoint.x && t.x + t.width >= selectedPoint.x && t.y <= selectedPoint.y && t.y + t.height >= selectedPoint.y;
		});
		if (!selectedTile) {
			console.warn('Could not find selected tile');
			return;
		}

		newShapes = newShapes.filter(s => s !== selectedShape);
		newShapes = newShapes.filter(s => s !== neighborShape);
		const newSelectedShape = {
			tiles: selectedShape.tiles.filter(t => t !== selectedTile),
			color: selectedShape.color,
			d: '',
			index: selectedShape.index
		};
		const newNeighborShape = {
			tiles: neighborShape.tiles.concat(selectedTile),
			color: neighborShape.color,
			d: '',
			index: neighborShape.index
		};
		newSelectedShape.d = generatePathFromTiles(newSelectedShape.tiles);
		newNeighborShape.d = generatePathFromTiles(newNeighborShape.tiles);
		newShapes.push(newSelectedShape);
		newShapes.push(newNeighborShape);
		shapes = newShapes;
	}

	function doIt() {
		const divisions = Math.floor(Math.random() * 5) + 3;
		for (let i = 0; i < divisions; i++) {
			subDivide();
		}
		const swaps = Math.floor(Math.random() * 3) + 3;
		for (let i = 0; i < swaps; i++) {
			swap();
		}
	}

	$inspect(shapes);

	$effect(() => {
		console.debug('Prompt29 mounted', svgId);
		setTimeout(() => {
			doIt();
		}, 0);
		return () => {
			console.debug('Prompt29 unmounted', svgId);
		};
	});
</script>

<SVGContainer>
	<svg
		role="graphics-object"
		id={svgId}
		viewBox="0 0 {WIDTH} {HEIGHT}"
		xmlns="http://www.w3.org/2000/svg"
		width={`${WIDTH}`}
		height={`${HEIGHT}`}
	>

		{#if DEBUG}
			{#each tiles as tile}
				<rect
					x={tile.x}
					y={tile.y}
					width={TILE_SIZE_X}
					height={TILE_SIZE_Y}
					fill="none"
					stroke="#FF0000"
					stroke-width="2"
				/>
			{/each}
		{/if}

		{#each shapes as shape}
			<path d={shape.d} fill={shape.color} opacity=".5" stroke="#000" stroke-width="20" />
			{#if DEBUG}
				{#each shape.tiles as tile}
					<text x={tile.x + 10} y={tile.y + 20} fill="#000" font-size="20">{`(${tile.x}, ${tile.y})`}</text>
				{/each}
			{/if}
		{/each}
	</svg>
</SVGContainer>
<hr />
<div>
	<input type="checkbox" bind:checked={DEBUG} aria-label="DEBUG flag on/off" /> debug
	{#if DEBUG}
		<button onclick={subDivide}>Subdivide</button>
		<button onclick={swap}>Swap</button>
		<button onclick={doIt}>Do It</button>
	{/if}

</div>

