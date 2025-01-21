<script lang="ts">
	// eslint-disable-next-line @typescript-eslint/ban-ts-comment
	//@ts-expect-error
	import mixbox from 'mixbox';
	import SVGContainer from '$lib/SVGContainer.svelte';

	const { svgId } = $props();

	const patterns = [
		function lissajous(elapsed: number, i: number) {
			const source = colors[i];
			const offset = offsets[i];
			const frequency = (1 + i * 0.5) * offset.freq;
			const phase = i * Math.PI / colors.length + offset.phase;

			source.x = remap(Math.sin(elapsed * frequency + phase), -1, 1, 0, WIDTH);
			source.y = remap(Math.sin(elapsed * frequency * 1.5 + phase), -1, 1, 0, HEIGHT);
		},

		function spiral(elapsed: number, i: number) {
			const source = colors[i];
			const offset = offsets[i];
			const angle = elapsed * offset.freq + (i * Math.PI * 2 / colors.length) + offset.phase;
			const radius = Math.sin(elapsed * 0.5 + offset.phase) * 0.4 * offset.scale + 0.6;

			source.x = remap(Math.cos(angle) * radius, -1, 1, 0, WIDTH);
			source.y = remap(Math.sin(angle) * radius, -1, 1, 0, HEIGHT);

		},

		function figure8(elapsed: number, i: number) {
			const source = colors[i];
			const offset = offsets[i];
			const t = elapsed * offset.freq + (i * Math.PI / colors.length) + offset.phase;

			source.x = remap(Math.sin(2 * t), -1, 1, 0, WIDTH);
			source.y = remap(Math.sin(t) * offset.scale, -1, 1, 0, HEIGHT);
		},

		function roseCurve(elapsed: number, i: number) {
			const source = colors[i];
			const offset = offsets[i];
			const k = (2 + (i % 3)) * offset.freq;
			const t = elapsed + (i * Math.PI / colors.length) + offset.phase;

			const r = Math.cos(k * t) * offset.scale;
			source.x = remap(r * Math.cos(t), -1, 1, 0, WIDTH);
			source.y = remap(r * Math.sin(t), -1, 1, 0, HEIGHT);
		},

		function parametric(elapsed: number, i: number) {
			const source = colors[i];
			const offset = offsets[i];
			const fx = (1 + i * 0.2) * offset.freq;
			const fy = (1.5 + i * 0.3) * offset.freq;

			source.x = remap(Math.sin(elapsed * fx + offset.phase) +
				Math.cos(elapsed * fy * 0.5), -2, 2, 0, WIDTH);
			source.y = remap(Math.sin(elapsed * fy + offset.phase) +
				Math.cos(elapsed * fx * 0.5), -2, 2, 0, HEIGHT);
		}
	];
	const WIDTH = 1080;
	const HEIGHT = 1080;
	// pigment colors from mixbox documentation
	const PIGMENT_COLORS = [
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

	type Tile = {
		x: number;
		y: number;
		size: number;
		index: number;
		color: string;
		closeness?: number[];
	};

	type ColorSource = {
		color: string;
		latent: number[];
		x: number;
		y: number;
		radius: number;
		index: number;
		animationFunction: (elapsed: number, index: number) => void;
	}

	let DEBUG = $state(false);
	const timeFactor = 2000 + Math.random() * 5000;
	let colors: ColorSource[] = $state([]);
	let tiles: Tile[] = $state([]);
	// Generate random offsets for each color at startup
	let offsets;
	let startTime: number;
	let animationId: number;

	$effect(() => {
		setTimeout(() => {
			colors = generateColorSources(Math.floor(Math.random() * 5) + 2);
			offsets = colors.map(() => ({
				phase: Math.random() * Math.PI * 2,
				scale: 0.5 + Math.random(),  // random scale between 0.5 and 1.5
				freq: 0.8 + Math.random() * 0.4  // random frequency multiplier between 0.8 and 1.2
			}));
			tiles = buildTiles(Math.floor(Math.random() * 12) + 3);
		});
		animationId = requestAnimationFrame(animate);
		return () => {
			cancelAnimationFrame(animationId);
		};
	});

	function buildTiles(dimension: number): Tile[] {
		const tiles: Tile[] = [];
		const size = WIDTH / dimension;
		for (let i = 0; i < dimension; i++) {
			for (let j = 0; j < dimension; j++) {
				const x = i * size;
				const y = j * size;
				const index = i * dimension + j;
				const color = '#000';
				tiles.push({ x, y, size, index, color });
			}
		}
		return tiles;
	}

	function generateColorSources(count: number): ColorSource[] {
		console.log('Generating', count, 'color sources');
		const indexes: number[] = [];
		const sources: ColorSource[] = [];
		for (let i = 0; i < count; i++) {
			const x = Math.random() * WIDTH;
			const y = Math.random() * HEIGHT;
			let index = Math.floor(Math.random() * PIGMENT_COLORS.length);
			while (indexes.includes(index)) {
				index = Math.floor(Math.random() * PIGMENT_COLORS.length);
			}
			indexes.push(index);
			const color = PIGMENT_COLORS[index];
			const latent = mixbox.rgbToLatent(color);
			const radius = Math.random() * (WIDTH / 2) + (WIDTH * .6);
			const patternIndex = Math.floor(Math.random() * patterns.length);
			const pattern = patterns[patternIndex];
			sources.push({ x, y, color, latent, radius, animationFunction: pattern, index: i });
			console.log('source', i, 'color', color, 'latent', latent, 'radius', radius, 'pattern', patternIndex);
		}
		return sources;
	}

	// return 0 to 1, 0 is farthest away (= WIDTH) and 1 is closest (= 0)
	function getClosenessFactor(tile: Tile, source: ColorSource): number {
		const dx = tile.x + (tile.size / 2) - source.x;
		const dy = tile.y + (tile.size / 2) - source.y;
		const distance = Math.sqrt(dx * dx + dy * dy) - (tile.size / 2);
		// if (distance > source.radius) return 0.05;
		return Math.max(0.001, 1 - distance / source.radius);//(source.radius * source.radius);
	}

	let osc = $state(0);

	function animate(timestamp: number) {
		if (!startTime) startTime = timestamp;
		const elapsed = (timestamp - startTime) / timeFactor;
		osc = remap(Math.sin(elapsed), -1, 1, 0.8, .95);
		const newTiles = [];
		for (let i = 0; i < tiles.length; i++) {
			const tile = { ...tiles[i] };
			let closeness = new Array(colors.length);
			for (let j = 0; j < colors.length; j++) {
				const source = colors[j];
				closeness[j] = getClosenessFactor(tile, source);
			}
			closeness = normalize(closeness, osc);
			const tileLatent = new Array(mixbox.LATENT_SIZE);
			for (var k = 0; k < tileLatent.length; k++) { // mix:
				let totalCloseness = 0;
				for (let j = 0; j < colors.length; j++) {
					totalCloseness += closeness[j] * colors[j].latent[k];
				}
				tileLatent[k] = totalCloseness;
			}
			tile.color = mixbox.latentToRgb(tileLatent);
			tile.closeness = closeness;
			newTiles.push(tile);
		}
		for (const color of colors) {
			color.animationFunction(elapsed, color.index);
		}
		tiles = newTiles;
		animationId = requestAnimationFrame(animate);
	}

	function normalize(array: number[], factor = 1) {
		const sum = array.reduce((acc, val) => acc + val, 0) * factor;
		// if (sum < 1) return array;
		return array.map(val => (val / sum));
	}

	function remap(value: number, fromMin: number, fromMax: number, toMin: number, toMax: number) {
		const normalized = (value - fromMin) / (fromMax - fromMin);
		return normalized * (toMax - toMin) + toMin;
	}
</script>

<SVGContainer>
	<svg
		role="graphics-object"
		id={svgId}
		viewBox="0 0 {WIDTH} {HEIGHT}"
		xmlns="http://www.w3.org/2000/svg"
		width={`${WIDTH}`}
		height={`${HEIGHT}`}
		filter="url(#noise)"
	>
		<defs>
			<filter id="noise" x="0%" y="0%" width="100%" height="100%">
				<feTurbulence
					type="fractalNoise"
					baseFrequency="1.5"
					numOctaves="4"
					seed="1"
					stitchTiles="stitch"
					result="noiseBase"
				/>

				<feTurbulence
					type="fractalNoise"
					baseFrequency="0.8"
					numOctaves="2"
					seed="2"
					result="noise2"
				/>

				<feComposite
					operator="arithmetic"
					in="noiseBase"
					in2="noise2"
					k1="0.5"
					k2="0.5"
					result="combinedNoise"
				/>

				<feGaussianBlur stdDeviation="0.7" result="softNoise" />

				<feComponentTransfer>
					<feFuncR type="linear" slope="1" intercept="0" />
					<feFuncG type="linear" slope="1" intercept="1" />
					<feFuncB type="linear" slope="1" intercept="1" />
				</feComponentTransfer>

				<feBlend
					in="SourceGraphic"
					in2="softNoise"
					mode="multiply"
					result="final"
				/>
			</filter>
		</defs>

		{#each tiles as tile}

			<rect x={tile.x} y={tile.y} width={tile.size} height={tile.size} fill={tile.color} stroke={tile.color} stroke-width="2"
						stroke-opacity="0.5"
			/>
		{/each}
		{#each tiles as tile}
			{#if DEBUG && tile.closeness}
				{#each tile.closeness as closeness, j}
					<circle cx={tile.x + 18} cy={tile.y + 20 +  20 * j} r={20 * closeness} fill={colors[j].color} stroke="#fff" stroke-width="2"
									stroke-opacity={closeness} />
					<text x={tile.x + 38} y={tile.y + 25 + 20 * j} fill="#fff" font-size="10" text-anchor="start"
								alignment-baseline="hanging">{closeness.toFixed(2)}</text>
				{/each}
			{/if}
		{/each}

		{#if !DEBUG}
			{#each colors as color, i}
				<rect x={10} y={10 + i * 20} width="20" height="20" fill={color.color} stroke-width="2" stroke="#fff" />
			{/each}
			<text x={12} y={15 + (colors.length + 1) * 20} fill="#FFF" stroke-width="2"
						font-weight="bold">{Math.round(remap(100 - (osc * 100), 5, 20, 0, 99))}</text>
		{:else}
			{#each colors as color}
				<circle cx={color.x} cy={color.y} r={color.radius} fill="none" stroke="#fff" />
				<circle cx={color.x} cy={color.y} r="10" fill={color.color} stroke="#000" stroke-width="3" />
			{/each}
		{/if}
	</svg>


</SVGContainer>
<hr />
<div>
	<input type="checkbox" bind:checked={DEBUG} aria-label="DEBUG flag on/off" /> debug
</div>
