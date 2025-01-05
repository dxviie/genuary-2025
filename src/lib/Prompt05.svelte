<script lang="ts">
	import SVGContainer from '$lib/SVGContainer.svelte';

	const { svgId } = $props();

	type Tile = {
		x: number;
		y: number;
		d: string;
	};

	const HEIGHT = 1080;
	const WIDTH = 1080;

	const TILE_WIDTH = 100;
	const TILE_HEIGHT = 50;

	const MAX_HORIZONTAL_TILE_COUNT = 10;
	let tiles: Tile[] = $state([]);

	$effect(() => {
		const rows = MAX_HORIZONTAL_TILE_COUNT * 2 - 1;
		let rowX = WIDTH / 2 - TILE_WIDTH / 2;
		let rowY = ((HEIGHT - ((TILE_HEIGHT / 2) * rows)) / 2) - TILE_HEIGHT / 2;
		let row = 0;
		let cols = 1;
		let peaked = false;
		let newTiles: Tile[] = [];
		while (cols > 0) {
			let x = rowX;
			let y = rowY;
			for (let c = 0; c < cols; c++) {
				newTiles.push({
					x: x,
					y: y,
					d: `M ${x} ${y} l ${TILE_WIDTH / 2} -${TILE_HEIGHT / 2} l ${TILE_WIDTH / 2} ${TILE_HEIGHT / 2} l -${TILE_WIDTH / 2} ${TILE_HEIGHT / 2} Z`
				});
				x += TILE_WIDTH;
			}

			if (peaked || cols === MAX_HORIZONTAL_TILE_COUNT) {
				peaked = true;
				cols--;
				rowX += TILE_WIDTH / 2;
			} else {
				cols++;
				rowX -= TILE_WIDTH / 2;
			}

			rowY += TILE_HEIGHT / 2;
			row++;

		}
		tiles = newTiles;
	});
	$inspect(tiles);
</script>

<SVGContainer>
	<svg
		id={svgId}
		viewBox="0 0 {WIDTH} {HEIGHT}"
		xmlns="http://www.w3.org/2000/svg"
		width={`${WIDTH}`}
		height={`${HEIGHT}`}
	>
		{#each tiles as tile, index}
			<path d={tile.d} stroke="#000" stroke-width="1" fill="none" />
			<text x={tile.x + TILE_WIDTH/2} y={tile.y} dominant-baseline="middle" text-anchor="middle" fill="#000">{index}</text>
		{/each}
	</svg>
</SVGContainer>