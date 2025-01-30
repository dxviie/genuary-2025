<script lang="ts">
	import SVGContainer from '$lib/SVGContainer.svelte';
	import { recorderState } from './ffmpegRecorder.svelte';

	const { svgId } = $props();

	type Tile = {
		x: number;
		y: number;
		index: number;
		headPath: string;
		leftPath: string;
		rightPath: string;
		buildingHeight: number;
		height: number;
	};

	const HEIGHT = 1080;
	const WIDTH = 1920;
	const MARGIN = -550;// (Math.random() * 1000) - 700;
	const STROKE_WIDTH = 2;

	const MAX_HORIZONTAL_TILE_COUNT = 25; //Math.floor(Math.random() * 28 + 2);
	const TILE_RATIO = 1; //Math.random() * 1.5 + .3;
	const TILE_WIDTH = (WIDTH - (2 * MARGIN)) / MAX_HORIZONTAL_TILE_COUNT;
	const TILE_HEIGHT = TILE_WIDTH * TILE_RATIO;
	const BASE_BUILDING_HEIGHT = 20;
	const MAX_BUILDING_HEIGHT = 20;// + Math.random() * 250;
	const MAX_WAVE_HEIGHT = 150;// + Math.random() * 50;

	const COLOR_ANIMATION_DURATION = 15000;// + Math.random() * 15000;

	const HEIGHT_ANIMATION_DURATION = 30000;
	const FREQUENCY = 0.1;//Math.random();

	let tiles: Tile[] = $state([]);
	let colors = $state({
		headColor: '#FFFFFF',
		leftColor: '#CCCCCC',
		rightColor: '#333333'
	});

	function buildHeadPath(tileX: number, tileY: number, height: number, tileWidth: number, tileHeight: number): string {
		return `M ${tileX} ${tileY - height} l ${tileWidth / 2} -${tileHeight / 2} l ${tileWidth / 2} ${tileHeight / 2} l -${tileWidth / 2} ${tileHeight / 2} Z`;
	}

	function buildLeftPath(tileX: number, tileY: number, height: number, tileWidth: number, tileHeight: number): string {
		return `M ${tileX} ${tileY} l 0 -${height} l ${tileWidth / 2} ${tileHeight / 2} l 0 ${height} Z`;
	}

	function buildRightPath(tileX: number, tileY: number, height: number, tileWidth: number, tileHeight: number): string {
		return `M ${tileX + tileWidth / 2} ${tileY + tileHeight / 2} l ${tileWidth / 2} -${tileHeight / 2} l 0 -${height} l -${tileWidth / 2} ${tileHeight / 2} Z`;
	}

	$effect(() => {
		recorderState.width = WIDTH;
		recorderState.height = HEIGHT;

		const rows = (MAX_HORIZONTAL_TILE_COUNT * 2) - 1;
		let rowX = WIDTH / 2 - TILE_WIDTH / 2;
		// (total height - grid height) / 2
		// + half tile height to adjust for the x,y point of the tiles being the leftmost point
		let rowY = (HEIGHT - ((rows + 1) * (TILE_HEIGHT / 2))) / 2 + TILE_HEIGHT / 2 + (MAX_BUILDING_HEIGHT + BASE_BUILDING_HEIGHT) / 2;
		console.debug('rows', rows, 'rowX', rowX, 'rowY', rowY);
		let row = 0;
		let cols = 1;
		let peaked = false;
		let newTiles: Tile[] = [];
		while (cols > 0) {
			let x = rowX;
			let y = rowY;
			for (let c = 0; c < cols; c++) {
				const height = MAX_BUILDING_HEIGHT;
				newTiles.push({
					buildingHeight: height,
					height: height,
					x: x,
					y: y,
					index: newTiles.length,
					headPath: buildHeadPath(x, y, height, TILE_WIDTH, TILE_HEIGHT),
					leftPath: buildLeftPath(x, y, height, TILE_WIDTH, TILE_HEIGHT),
					rightPath: buildRightPath(x, y, height, TILE_WIDTH, TILE_HEIGHT)
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

	$effect(() => {
		colorStartTime = 0;
		colorAnimationId = requestAnimationFrame(Math.random() > 0.5 ? animateHue : animateGray);

		if (animationId) {
			cancelAnimationFrame(animationId);
		}
		startTime = 0;
		animationId = requestAnimationFrame(animateWave);

		return () => {
			if (colorAnimationId) cancelAnimationFrame(colorAnimationId);
			if (animationId) {
				cancelAnimationFrame(animationId);
			}
		};
	});

	/**********************************************
	 * COLOR ANIMATION STUFF
	 * Courtesy of Claude
	 * ********************************************/

	let colorAnimationId: number;
	let colorStartTime: number;

	const LEFT_SAT = Math.random() * 60 + 30;
	const HUE_HEAD = Math.random() * 360;
	const HUE_LEFT = Math.random() * 360;
	const HUE_RIGHT = Math.random() * 360;

	function animateHue(timestamp: number) {
		if (!colorStartTime) colorStartTime = timestamp;
		let elapsed = timestamp - colorStartTime;
		if (recorderState.recording) {
			// * 2.5 as we're usually drawing at 60+ fps
			elapsed = (recorderState.frame / (recorderState.fps * 2.5));
		}
		const progress = (elapsed % COLOR_ANIMATION_DURATION) / COLOR_ANIMATION_DURATION;
		const angle = progress * Math.PI * 2;

		// Calculate HSL values for each side
		const head = (Math.sin(angle) + 1) / 2;
		const left = (Math.sin(angle + (Math.PI * 2 / 3)) + 1) / 2;
		const right = (Math.sin(angle + (Math.PI * 4 / 3)) + 1) / 2;

		colors = {
			headColor: hslToHex(HUE_HEAD + head, 70, 60),
			leftColor: hslToHex(HUE_LEFT + left, LEFT_SAT, 60),
			rightColor: hslToHex(HUE_RIGHT + right, 100 - LEFT_SAT, 60)
		};

		colorAnimationId = requestAnimationFrame(animateHue);
	}

	function hslToHex(h: number, s: number, l: number): string {
		h /= 360;
		s /= 100;
		l /= 100;
		let r, g, b;

		if (s === 0) {
			r = g = b = l;
		} else {
			const hue2rgb = (p: number, q: number, t: number) => {
				if (t < 0) t += 1;
				if (t > 1) t -= 1;
				if (t < 1 / 6) return p + (q - p) * 6 * t;
				if (t < 1 / 2) return q;
				if (t < 2 / 3) return p + (q - p) * (2 / 3 - t) * 6;
				return p;
			};

			const q = l < 0.5 ? l * (1 + s) : l + s - l * s;
			const p = 2 * l - q;
			r = hue2rgb(p, q, h + 1 / 3);
			g = hue2rgb(p, q, h);
			b = hue2rgb(p, q, h - 1 / 3);
		}

		const toHex = (x: number) => {
			const hex = Math.round(x * 255).toString(16);
			return hex.length === 1 ? '0' + hex : hex;
		};

		return `#${toHex(r)}${toHex(g)}${toHex(b)}`;
	}

	function animateGray(timestamp: number) {
		if (!colorStartTime) colorStartTime = timestamp;
		let elapsed = timestamp - colorStartTime;
		if (recorderState.recording) {
			// * 2.5 as we're usually drawing at 60+ fps
			elapsed = (recorderState.frame / (recorderState.fps * 2.5));
		}
		const progress = (elapsed % COLOR_ANIMATION_DURATION) / COLOR_ANIMATION_DURATION;

		// Calculate the position of the "light" (0 to 1)
		const angle = progress * Math.PI * 2;

		// Calculate brightness for each side using sine waves
		// Offset each side by 120 degrees (2π/3) for even distribution
		const headBrightness = (Math.sin(angle) + 1) / 2;
		const leftBrightness = (Math.sin(angle + (Math.PI * 2 / 3)) + 1) / 2;
		const rightBrightness = (Math.sin(angle + (Math.PI * 4 / 3)) + 1) / 2;

		// Convert brightness to hex colors
		colors = {
			headColor: brightnessToHex(headBrightness),
			leftColor: brightnessToHex(leftBrightness),
			rightColor: brightnessToHex(rightBrightness)
		};

		colorAnimationId = requestAnimationFrame(animateGray);
	}

	function brightnessToHex(brightness: number): string {
		// Convert brightness (0-1) to hex color (gray scale)
		const value = Math.round(brightness * 255);
		const hex = value.toString(16).padStart(2, '0');
		return `#${hex}${hex}${hex}`;
	}

	/*****************************************************
	 * 	HEIGHT ANIMATION STUFF
	 * 	Courtesy of Claude
	 *  *************************************************/

	let animationId: number;
	let startTime: number;

	function animateWave(timestamp: number) {
		if (!startTime) startTime = timestamp;
		const elapsed = timestamp - startTime;
		const progress = (elapsed % HEIGHT_ANIMATION_DURATION) / HEIGHT_ANIMATION_DURATION;

		tiles = tiles.map((tile, index) => {
			// Create a wave pattern based on tile position

			const waveX = (tile.index) * 0.5; // Diagonal wave
			const wave = Math.sin(2 * Math.PI * (waveX * FREQUENCY + progress));
			const newHeight = tile.buildingHeight + (wave + 1) * MAX_WAVE_HEIGHT / 2;
			const headPath = buildHeadPath(tile.x, tile.y, newHeight, TILE_WIDTH, TILE_HEIGHT);
			const leftPath = buildLeftPath(tile.x, tile.y, newHeight, TILE_WIDTH, TILE_HEIGHT);
			const rightPath = buildRightPath(tile.x, tile.y, newHeight, TILE_WIDTH, TILE_HEIGHT);

			return {
				...tile,
				height: newHeight,
				headPath: headPath,
				leftPath: leftPath,
				rightPath: rightPath
			};
		});

		animationId = requestAnimationFrame(animateWave);
	}

</script>

<!--<SVGContainer>-->
<svg
	id={svgId}
	viewBox="0 0 {WIDTH} {HEIGHT}"
	xmlns="http://www.w3.org/2000/svg"
	width={`${WIDTH}px`}
	height={`${HEIGHT}px`}
	class="svg-edit"
>
	<rect x={0} y={0} width={WIDTH} height={HEIGHT} fill="#000"></rect>
	{#each tiles as tile, index}
		{#if tile.x < WIDTH / 2}
			<path d={tile.rightPath} stroke={colors.rightColor} stroke-width={STROKE_WIDTH} fill={colors.rightColor} />
			<path d={tile.leftPath} stroke={colors.leftColor} stroke-width={STROKE_WIDTH} fill={colors.leftColor} />
			<path d={tile.headPath} stroke={colors.headColor} stroke-width={STROKE_WIDTH} fill={colors.headColor} />
		{:else}
			<path d={tile.leftPath} stroke={colors.leftColor} stroke-width={STROKE_WIDTH} fill={colors.leftColor} />
			<path d={tile.rightPath} stroke={colors.rightColor} stroke-width={STROKE_WIDTH} fill={colors.rightColor} />
			<path d={tile.headPath} stroke={colors.headColor} stroke-width={STROKE_WIDTH} fill={colors.headColor} />
		{/if}
	{/each}
</svg>
<!--</SVGContainer>-->

<style>
    .svg-edit {
        width: 1280px;
        height: 720px;
    }
</style>