<script lang="ts">
	import SVGContainer from '$lib/SVGContainer.svelte';

	const { svgId } = $props();

	type Tile = {
		x: number;
		y: number;
		headPath: string;
		leftPath: string;
		rightPath: string;
		height: number;
	};

	const HEIGHT = 1080;
	const WIDTH = 1080;
	const MARGIN = 50;
	const STROKE_WIDTH = 2;

	const MAX_HORIZONTAL_TILE_COUNT = 20;
	const TILE_RATIO = 0.5;
	const TILE_WIDTH = (WIDTH - (2 * MARGIN)) / MAX_HORIZONTAL_TILE_COUNT;
	const TILE_HEIGHT = TILE_WIDTH * TILE_RATIO;
	const MAX_BUILDING_HEIGHT = 500;

	let tiles: Tile[] = $state([]);
	let colors = $state({
		headColor: '#FFFFFF',
		leftColor: '#CCCCCC',
		rightColor: '#333333'
	});

	function buildHeadPath(tileX: number, tileY: number, height: number, tileWidth: number, tileHeight: number): string {
		return `M ${tileX} ${tileY - height} l ${tileWidth / 2} -${tileHeight / 2} l ${tileWidth / 2} ${tileHeight / 2} l -${tileWidth / 2} ${tileHeight / 2} Z`;
	}

	$effect(() => {
		const rows = (MAX_HORIZONTAL_TILE_COUNT * 2) - 1;
		let rowX = WIDTH / 2 - TILE_WIDTH / 2;
		// (total height - grid height) / 2
		// + half tile height to adjust for the x,y point of the tiles being the leftmost point
		let rowY = (HEIGHT - ((rows + 1) * (TILE_HEIGHT / 2))) / 2 + TILE_HEIGHT / 2 + MAX_BUILDING_HEIGHT / 2;
		console.debug('rows', rows, 'rowX', rowX, 'rowY', rowY);
		let row = 0;
		let cols = 1;
		let peaked = false;
		let newTiles: Tile[] = [];
		while (cols > 0) {
			let x = rowX;
			let y = rowY;
			for (let c = 0; c < cols; c++) {
				const height = Math.floor(Math.random() * MAX_BUILDING_HEIGHT);
				newTiles.push({
					height: height,
					x: x,
					y: y,
					headPath: buildHeadPath(x, y, height, TILE_WIDTH, TILE_HEIGHT),
					leftPath: `M ${x} ${y} l 0 -${height} l ${TILE_WIDTH / 2} ${TILE_HEIGHT / 2} l 0 ${height} Z`,
					rightPath: `M ${x + TILE_WIDTH / 2} ${y + TILE_HEIGHT / 2} l ${TILE_WIDTH / 2} -${TILE_HEIGHT / 2} l 0 -${height} l -${TILE_WIDTH / 2} ${TILE_HEIGHT / 2} Z`
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
	let animationId: number;
	let startTime: number;
	const ANIMATION_DURATION = 60000;

	function animateHue(timestamp: number) {
		if (!startTime) startTime = timestamp;
		const elapsed = timestamp - startTime;
		const progress = (elapsed % ANIMATION_DURATION) / ANIMATION_DURATION;
		const angle = progress * Math.PI * 2;

		// Calculate HSL values for each side
		const head = (Math.sin(angle) + 1) / 2;
		const left = (Math.sin(angle + (Math.PI * 2 / 3)) + 1) / 2;
		const right = (Math.sin(angle + (Math.PI * 4 / 3)) + 1) / 2;

		colors = {
			headColor: hslToHex(head * 360, 70, 60),
			leftColor: hslToHex(left * 360, 70, 60),
			rightColor: hslToHex(right * 360, 70, 60)
		};

		animationId = requestAnimationFrame(animateHue);
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
		if (!startTime) startTime = timestamp;
		const elapsed = timestamp - startTime;
		const progress = (elapsed % ANIMATION_DURATION) / ANIMATION_DURATION;

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

		animationId = requestAnimationFrame(animateGray);
	}

	function brightnessToHex(brightness: number): string {
		// Convert brightness (0-1) to hex color (gray scale)
		const value = Math.round(brightness * 255);
		const hex = value.toString(16).padStart(2, '0');
		return `#${hex}${hex}${hex}`;
	}

	$effect(() => {
		startTime = 0;
		animationId = requestAnimationFrame(Math.random() > 0.5 ? animateHue : animateGray);
		return () => {
			if (animationId) cancelAnimationFrame(animationId);
		};
	});
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
			<path d={tile.leftPath} stroke={colors.leftColor} stroke-width={STROKE_WIDTH} fill={colors.leftColor} />
			<path d={tile.rightPath} stroke={colors.rightColor} stroke-width={STROKE_WIDTH} fill={colors.rightColor} />
			<path d={tile.headPath} stroke={colors.headColor} stroke-width={STROKE_WIDTH} fill={colors.headColor} />
		{/each}
	</svg>
</SVGContainer>