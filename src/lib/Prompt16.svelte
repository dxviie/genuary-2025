<script lang="ts">
	// eslint-disable-next-line @typescript-eslint/ban-ts-comment
	//@ts-expect-error
	import mixbox from 'mixbox';
	import SVGContainer from '$lib/SVGContainer.svelte';

	const WIDTH = 1080;
	const HEIGHT = 1080;
	// pigment colors from mixbox documentation
	const colors = [
		'rgb(254, 236, 0)',   // Cadmium Yellow
		'rgb(252, 211, 0)',   // Hansa Yellow
		'rgb(255, 105, 0)',   // Cadmium Orange
		'rgb(255, 39, 2)',    // Cadmium Red
		'rgb(128, 2, 46)',    // Quinacridone Magenta
		'rgb(78, 0, 66)',     // Cobalt Violet
		'rgb(25, 0, 89)',     // Ultramarine Blue
		'rgb(0, 33, 133)',    // Cobalt Blue
		'rgb(13, 27, 68)',    // Phthalo Blue
		'rgb(0, 60, 50)',     // Phthalo Green
		'rgb(7, 109, 22)',    // Permanent Green
		'rgb(107, 148, 4)',   // Sap Green
		'rgb(123, 72, 0)'     // Burnt Sienna
	];

	const rgb1 = colors[Math.floor(Math.random() * colors.length)];
	let rgb2 = colors[Math.floor(Math.random() * colors.length)];

	const { svgId } = $props();
	let tiles = $state(buildTiles(8));

	type Tile = {
		x: number;
		y: number;
		size: number;
		index: number;
		color: string;
	};

	function buildTiles(dimension: number): Tile[] {
		const tiles: Tile[] = [];
		const size = WIDTH / dimension;
		const tileCount = dimension * dimension;
		for (let i = 0; i < dimension; i++) {
			for (let j = 0; j < dimension; j++) {
				const x = i * size;
				const y = j * size;
				const index = i * dimension + j;
				const color = mixbox.lerp(rgb1, rgb2, index / tileCount);
				tiles.push({ x, y, size, index, color });
			}
		}
		return tiles;
	}

	let startTime: number;
	let animationId: number;

	$effect(() => {
		animationId = requestAnimationFrame(animate);
		return () => {
			cancelAnimationFrame(animationId);
		};
	});

	function animate(timestamp: number) {
		if (rgb1 === rgb2) {
			rgb2 = colors[Math.floor(Math.random() * colors.length)];
		}
		if (!startTime) startTime = timestamp;
		const elapsed = (timestamp - startTime) / 1000;
		const osc = (Math.sin(elapsed)) * (tiles.length * 1.2);
		for (let i = 0; i < tiles.length; i++) {
			const tile = tiles[i];
			tile.color = mixbox.lerp(rgb1, rgb2, (tile.index + osc) / tiles.length);
		}
		tiles = [...tiles];
		// console.log('tiles', tiles);
		animationId = requestAnimationFrame(animate);
	}

</script>

<SVGContainer>
	<svg
		id={svgId}
		viewBox="0 0 {WIDTH} {HEIGHT}"
		xmlns="http://www.w3.org/2000/svg"
		width={`${WIDTH}`}
		height={`${HEIGHT}`}
	>
		{#each tiles as tile}
			<rect x={tile.x} y={tile.y} width={tile.size} height={tile.size} fill={tile.color} />
			<!--			<text x={tile.x + tile.size / 2} y={tile.y + tile.size / 2} fill="#FFF" font-size="20" text-anchor="middle"-->
			<!--						alignment-baseline="middle">{tile.index}</text>-->
		{/each}
	</svg>

</SVGContainer>