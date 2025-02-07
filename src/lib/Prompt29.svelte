<script lang="ts">
	import SVGContainer from '$lib/SVGContainer.svelte';
	import { recorderState } from '$lib/ffmpegRecorder.svelte';

	const { svgId } = $props();
	let DEBUG = $state(false);
	let PLAY = $state(true);

	const WIDTH = 1080;
	const HEIGHT = 1080;
	const TILE_COUNT_X = 8;
	const TILE_SIZE_X = WIDTH / TILE_COUNT_X;
	const TILE_COUNT_Y = Math.round(HEIGHT / TILE_SIZE_X);
	const TILE_SIZE_Y = HEIGHT / TILE_COUNT_Y;

	type Tile = {
		x: number;
		y: number;
		width: number;
		height: number;
		index: number;
		shape: number; // index of the shape this tile belongs to
	}

	type Shape = {
		tiles: Tile[];
		d: string;
		color: string;
		index: number;
		swapCount: number;
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
		const pathD = `M ${path[0].x} ${path[0].y} ` +
			path.slice(1).map(p => `L ${p.x} ${p.y}`).join(' ') +
			' Z';

		return roundAndInsetPath(tiles, pathD, 10 + Math.random() * 20, 10 + Math.random() * 20);
	}

	function arePointsEqual(p1: Point, p2: Point): boolean {
		return Math.abs(p1.x - p2.x) < 0.001 && Math.abs(p1.y - p2.y) < 0.001;
	}

	const rotation = Math.random() * 137.508;
	let shapes: Shape[] = $state([buildInitialShape()]);

	function buildInitialShape(): Shape {
		let tiles: Tile[] = Array.from({ length: TILE_COUNT_X * TILE_COUNT_Y }, (_, index) => {
			const x = (index % TILE_COUNT_X) * TILE_SIZE_X;
			const y = (Math.floor(index / TILE_COUNT_X)) * TILE_SIZE_Y;
			return { x, y, index, width: TILE_SIZE_X, height: TILE_SIZE_Y, shape: 0 };
		});
		return { tiles, d: generatePathFromTiles(tiles), color: getColor(0), index: 0, swapCount: 0 };
	}

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
			let newTopTiles = topTiles.map(t => ({ ...t, shape: selectedShape.index }));
			let newBottomTiles = bottomTiles.map(t => ({ ...t, shape: shapes.length }));
			newShapes.push({
				tiles: newTopTiles,
				d: generatePathFromTiles(newTopTiles),
				color: selectedShape.color,
				index: selectedShape.index,
				swapCount: selectedShape.swapCount
			});
			newShapes.push({
				tiles: newBottomTiles,
				d: generatePathFromTiles(newBottomTiles),
				color: getColor(shapes.length),
				index: shapes.length,
				swapCount: 0
			});
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
			let newLeftTiles: Tile[] = leftTiles.map(t => ({ ...t, shape: selectedShape.index }));
			let newRightTiles: Tile[] = rightTiles.map(t => ({ ...t, shape: shapes.length }));
			newShapes.push({
				tiles: newLeftTiles,
				d: generatePathFromTiles(newLeftTiles),
				color: selectedShape.color,
				index: selectedShape.index,
				swapCount: selectedShape.swapCount
			});
			newShapes.push({
				tiles: newRightTiles,
				d: generatePathFromTiles(newRightTiles),
				color: getColor(shapes.length),
				index: shapes.length,
				swapCount: 0
			});
		}

		if (newShapes.filter(s => s.tiles.length === 0).length > 0) {
			// TODO when our shapes become too small (+- 2x2), the splitting fails
			console.error('Empty shapes found', newShapes);
			return;
		}
		shapes = newShapes;
	}

	function getNeighboringIndecesForTileIndex(index: number): Set<number> {
		const x = index % TILE_COUNT_X;
		const y = Math.floor(index / TILE_COUNT_X);
		const neighbors = new Set<number>();
		if (x > 0) neighbors.add(index - 1);
		if (x < TILE_COUNT_X - 1) neighbors.add(index + 1);
		if (y > 0) neighbors.add(index - TILE_COUNT_X);
		if (y < TILE_COUNT_Y - 1) neighbors.add(index + TILE_COUNT_X);
		return neighbors;
	}

	function getNeighboringTilesForTile(tile: Tile, includeOwn = false): Set<Tile> {
		const neighbors = getNeighboringIndecesForTileIndex(tile.index);
		const tiles = new Set<Tile>();
		shapes.forEach(shape => {
			shape.tiles.forEach(t => {
				if (!includeOwn && shape.index === tile.shape) return;
				if (neighbors.has(t.index)) {
					tiles.add(t);
				}
			});
		});
		return tiles;
	}

	function getNeighboringTilesForShape(shape: Shape): Tile[] {
		const neighbors = new Set<Tile>();
		shape.tiles.forEach(tile => {
			const tileNeighbors = getNeighboringTilesForTile(tile);
			tileNeighbors.forEach(t =>
				neighbors.add(t)
			);
		});
		return Array.from(neighbors);
	}

	function canSwapTile(tile: Tile | undefined): boolean {
		if (!tile) return false;
		const neighbors = getNeighboringTilesForTile(tile, true);
		let count = 0;
		neighbors.forEach(t => {
			if (t.shape === tile.shape) count++;
		});
		return count > 2;
	}

	const MAX_SWAPS = 3;

	function swap() {
		let newShapes = shapes.filter(s => true);
		let eligibleShapes = newShapes.filter(s => s.swapCount < MAX_SWAPS);
		let selectedShape = eligibleShapes[Math.floor(Math.random() * eligibleShapes.length)];
		if (!selectedShape) {
			console.warn('No shape found with less than', MAX_SWAPS, 'swaps');
			return;
		}
		if (selectedShape.tiles.length < 2) {
			console.warn('Cannot subdivide a shape with less than 2 tiles');
			return;
		}
		console.log('Selected shape', $state.snapshot(selectedShape));

		let neighbors = getNeighboringTilesForShape(selectedShape);
		console.log('Neighbors', $state.snapshot(neighbors));

		let index = 0;
		let selectedTile;
		let isValidTile = false;
		while ((!selectedTile || !isValidTile) && index < neighbors.length) {
			selectedTile = neighbors[index++];
			isValidTile = canSwapTile(selectedTile);
		}
		if (!selectedTile || !isValidTile) {
			console.warn('No valid tile found to swap', $state.snapshot(selectedTile), isValidTile);
			return;
		}
		console.log('Neighbor tile', $state.snapshot(selectedTile));

		let neighborShape = shapes.find(s => s.index === selectedTile.shape);
		if (!neighborShape) {
			console.warn('No neighbor shape found');
			return;
		}

		newShapes = newShapes.filter(s => s !== selectedShape);
		newShapes = newShapes.filter(s => s.index !== selectedTile.shape);
		const newSelectedShape = {
			tiles: selectedShape.tiles.concat(selectedTile).map(t => ({ ...t, shape: selectedShape.index })),
			color: selectedShape.color,
			d: '',
			index: selectedShape.index,
			swapCount: selectedShape.swapCount + 1
		};
		const newNeighborShape = {
			tiles: neighborShape.tiles.filter(t => t !== selectedTile).map(t => ({ ...t, shape: neighborShape.index })),
			color: neighborShape.color,
			d: '',
			index: neighborShape.index,
			swapCount: neighborShape.swapCount + 1
		};
		newSelectedShape.d = generatePathFromTiles(newSelectedShape.tiles);
		newNeighborShape.d = generatePathFromTiles(newNeighborShape.tiles);
		newShapes.push(newSelectedShape);
		newShapes.push(newNeighborShape);
		shapes = newShapes;
	}

	function doIt() {
		const divisions = Math.floor(Math.random() * 3) + 2;
		for (let i = 0; i < divisions; i++) {
			subDivide();
		}
		const swaps = Math.floor(Math.random() * 3) + 3;
		for (let i = 0; i < swaps; i++) {
			swap();
		}
	}

	export function roundAndInsetPath(
		pathTiles: Tile[],
		pathData: string,
		radius = 20,
		inset = 0
	): string {
		// Parse the path into points
		const points = pathData
			.split(/(?=[MLZ])/g)
			.filter((cmd) => cmd.trim())
			.map((cmd) => {
				const parts = cmd.trim().split(/\s+/);
				if (parts[0] === 'Z') return null;
				return {
					x: parseFloat(parts[1]),
					y: parseFloat(parts[2])
				};
			})
			.filter((p) => p !== null);

		// Close the path by adding the first point at the end if needed
		if (points[0].x !== points[points.length - 1].x || points[0].y !== points[points.length - 1].y) {
			points.push({ ...points[0] });
		}
		// console.debug('extracted points', points);

		// Apply inset if specified
		if (inset > 0) {
			const len = points.length - 1; // -1 because last point is same as first
			const newPoints = points.map((_, i) => ({ ...points[i] })); // Create a copy to avoid modifying while iterating

			for (let i = 0; i < len; i++) {
				const prev = points[(i - 1 + len) % len];
				const current = points[i];
				const next = points[(i + 1) % len];

				// Calculate vectors to previous and next points
				const toPrev = { x: prev.x - current.x, y: prev.y - current.y };
				const toNext = { x: next.x - current.x, y: next.y - current.y };

				// Normalize vectors
				const toPrevLen = Math.hypot(toPrev.x, toPrev.y);
				const toNextLen = Math.hypot(toNext.x, toNext.y);

				if (toPrevLen === 0 || toNextLen === 0) continue;

				toPrev.x /= toPrevLen;
				toPrev.y /= toPrevLen;
				toNext.x /= toNextLen;
				toNext.y /= toNextLen;

				// Calculate normal vector (average of normals from both segments)
				const normal = {
					x: (toPrev.x + toNext.x) / 2,
					y: (toPrev.y + toNext.y) / 2
				};

				if (normal.x === 0 && normal.y === 0) {
					if (prev.x === current.x && next.x === current.x) {
						normal.x = -1;
						normal.y = 0;
					} else if (prev.y === current.y && next.y === current.y) {
						normal.x = 0;
						normal.y = -1;
					}
				}
				// console.debug('toPrev', toPrev, 'toNext', toNext, 'normal', normal);
				// Normalize the normal vector
				const normalLen = Math.hypot(normal.x, normal.y);
				if (normalLen > 0) {
					normal.x /= normalLen;
					normal.y /= normalLen;

					const newX = normal.x * inset;
					const newY = normal.y * inset;
					// check if new point is inside any of the tiles
					const insideTile = pathTiles.some((tile) => {
						return (
							current.x + newX >= tile.x &&
							current.x + newX <= tile.x + tile.width &&
							current.y + newY >= tile.y &&
							current.y + newY <= tile.y + tile.height
						);
					});
					if (!insideTile) {
						normal.x *= -1;
						normal.y *= -1;
					}

					// Apply inset along the normal direction
					newPoints[i].x += normal.x * inset;
					newPoints[i].y += normal.y * inset;
				}
			}

			// Update the last point to match the first point
			newPoints[len] = { ...newPoints[0] };

			// Replace the original points with the inset points
			points.length = 0;
			points.push(...newPoints);
		}

		// Generate rounded path
		let path = '';
		const len = points.length - 1; // -1 because last point is same as first

		for (let i = 0; i < len; i++) {
			const current = points[i];
			const next = points[(i + 1) % len];
			const prev = points[(i - 1 + len) % len];

			// Calculate direction vectors
			const toPrev = { x: prev.x - current.x, y: prev.y - current.y };
			const toNext = { x: next.x - current.x, y: next.y - current.y };

			// Normalize vectors
			const toPrevLen = Math.hypot(toPrev.x, toPrev.y);
			const toNextLen = Math.hypot(toNext.x, toNext.y);

			if (toPrevLen === 0 || toNextLen === 0) continue;

			toPrev.x /= toPrevLen;
			toPrev.y /= toPrevLen;
			toNext.x /= toNextLen;
			toNext.y /= toNextLen;

			// Calculate corner radius for this point
			const actualRadius = Math.min(radius, toPrevLen / 2, toNextLen / 2);

			// Calculate control points
			const cp1 = {
				x: current.x + toPrev.x * actualRadius,
				y: current.y + toPrev.y * actualRadius
			};

			const cp2 = {
				x: current.x + toNext.x * actualRadius,
				y: current.y + toNext.y * actualRadius
			};

			// Start path if first point
			if (i === 0) {
				path += `M ${cp1.x} ${cp1.y}`;
			}

			// Add the rounded corner
			path += ` L ${cp1.x} ${cp1.y} Q ${current.x} ${current.y} ${cp2.x} ${cp2.y}`;
		}

		return path + ' Z';
	}

	$inspect(shapes);

	let animationId: number;
	let lastFrameTimestamp = 0;
	const FPS = 3;
	const maxFrameCount = 30;
	let frameCount = $state(0);

	function animate() {
		const now = performance.now();
		if ((recorderState.recording || PLAY) && frameCount < maxFrameCount && (!lastFrameTimestamp || (now - lastFrameTimestamp) > (1000 / FPS))) {
			setTimeout(() => frameCount++, 0);
			if (Math.random() > 0.5) {
				subDivide();
			} else {
				swap();
			}
			lastFrameTimestamp = now;
		}
		animationId = requestAnimationFrame(animate);
	}

	$effect(() => {
		console.debug('Prompt29 mounted', svgId);
		recorderState.maxSeconds = 13;
		animationId = requestAnimationFrame(animate);
		return () => {
			console.debug('Prompt29 unmounted', svgId);
			if (animationId > 0) {
				cancelAnimationFrame(animationId);
			}
		};
	});

	const bg = Math.random() > 0.5 ? '#000' : '#FFF';
</script>

<SVGContainer>
	<svg
		role="graphics-object"
		id={svgId}
		viewBox={`0 0 ${WIDTH} ${HEIGHT}`}
		xmlns="http://www.w3.org/2000/svg"
		width={WIDTH}
		height={HEIGHT}
	>
		<rect x="0" y="0" width={WIDTH} height={HEIGHT} fill={bg} />
		{#each shapes as shape}
			<path d={shape.d} fill={shape.color} opacity=".5" stroke={bg} stroke-width="10" />
			{#if DEBUG}
				{#if shape.tiles && shape.tiles.length > 0}
					<text x={shape.tiles[0].x + shape.tiles[0].width/3} y={shape.tiles[0].y + shape.tiles[0].height / 2} fill="#000"
								font-size="50">{shape.index}</text>
				{/if}
				{#each shape.tiles as tile}
					<rect
						x={tile.x}
						y={tile.y}
						width={TILE_SIZE_X}
						height={TILE_SIZE_Y}
						fill="none"
						stroke="#FF0000"
						stroke-width="2"
					/>
					<text x={tile.x + 10} y={tile.y + 20} fill="#000" font-size="20">{`(${tile.x}, ${tile.y})`}</text>
					<text x={tile.x + 10} y={tile.y + 40} fill="#000" font-size="20">{tile.index}</text>
				{/each}
			{/if}
		{/each}
<!--		<text x="50" y="70" fill="#FFF" stroke="#FFF" stroke-width="2" font-size="20">{`${frameCount}/${maxFrameCount}`}</text>-->
	</svg>
</SVGContainer>
<hr />
<div>
	<input type="checkbox" bind:checked={PLAY} aria-label="PLAY flag on/off" /> play
	<input type="checkbox" bind:checked={DEBUG} aria-label="DEBUG flag on/off" /> debug

	<button onclick={subDivide}>Subdivide</button>
	<button onclick={swap}>Swap</button>
	<button onclick={doIt}>Do It</button>
</div>

