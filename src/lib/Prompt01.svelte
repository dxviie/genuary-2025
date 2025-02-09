<script lang="ts">
	import SVGContainer from '$lib/SVGContainer.svelte';

	const { svgId } = $props();

	const HEIGHT = 1080;
	const WIDTH = 1080;
	const ZERO = performance.now();

	type TransformFunction = (time: DOMHighResTimeStamp) => number;
	type Line = { x1: number; y1: number; x2: number; y2: number; width: number; };
	type Mask = { id: string; cx: number; cy: number; r: number; type: 'circle' | 'rect' | string };
	type LineGroup = {
		index: number;
		mask: Mask;
		lines: Line[];
		fx: TransformFunction;
		fy: TransformFunction;
	};

	let styles = $state(['']);
	const R = 250;
	let animationFrame = -1;
	const LINE_GROUPS = $state(generateLineGroups());

	function generateLineGroups() {
		const minWidth = 0.5;
		const maxWidth = 10;
		const minSpacing = 5 + (Math.random() * 15);
		const maxSpacing = 50;
		const allowCrossing = Math.random() > 0.8;
		let horizontal = Math.random() > 0.5;
		let currentWidth = Math.random() * (maxWidth - minWidth) + minWidth;
		let currentSpacing = Math.random() * (maxSpacing - minSpacing) + minSpacing;
		let availableSpace = currentSpacing;
		let timeRatio = Math.random() * 2000;
		let phase = 0;
		let phaseDelta = Math.random() * 2 * Math.PI;
		let start = 0;
		const groups: LineGroup[] = [];
		while (availableSpace > 0) {
			const maskCx = WIDTH / 2 + (Math.random() * 2 - 1) * (R / 2);
			const maskCy = HEIGHT / 2 + (Math.random() * 2 - 1) * (R);
			const r = Math.random() * R / 2 + R / 2;
			groups.push({
				index: groups.length,
				mask: createMask(`mask-${groups.length}`, maskCx, maskCy, r),
				lines: generateLines(horizontal, currentSpacing, currentWidth, start),
				fx: createAnimationFunction(Math.sin, timeRatio * phase, 250, phase),
				fy: createAnimationFunction(Math.cos, timeRatio, 5, phase)
			});
			availableSpace -= currentWidth;
			start += currentWidth;
			phase += phaseDelta;
			if (allowCrossing) {
				horizontal = Math.random() > 0.5;
			}

			currentWidth = Math.random() * (maxWidth - minWidth) + minWidth;
			currentSpacing = Math.random() * (availableSpace - minSpacing) + minSpacing;
		}
		console.debug('Generated line groups:', groups);
		return groups;
	}

	function generateLines(horizontal = true, spacing = 10, width = 1, start = 0, stop = Number.MAX_VALUE) {
		const lines = [];
		const end = stop < Number.MAX_VALUE ? stop : horizontal ? WIDTH : HEIGHT;
		for (let i = start; i < end; i += (spacing + width)) {
			lines.push({
				x1: horizontal ? 0 : i,
				y1: horizontal ? i : 0,
				x2: horizontal ? WIDTH : i,
				y2: horizontal ? i : HEIGHT,
				width: width
			});
		}
		return lines;
	}

	function createMask(id: string, cx: number, cy: number, r: number) {
		return { id, cx, cy, r, type: Math.random() > 0.8 ? 'circle' : 'rect' };
	}

	function createAnimationFunction(f = Math.sin, timeRatio = 1000, amplitude = R, phase = 0) {
		return (t: DOMHighResTimeStamp) => (f(t / timeRatio) * amplitude) + phase;
	}

	function createTransform(time: DOMHighResTimeStamp, fx: TransformFunction, fy: TransformFunction) {
		const cx = fx(time);
		const cy = fy(time);
		return `transform: translate(${cx}px, ${cy}px)`;
	}

	const FPS = 30;
	const frameMillis = 1000 / FPS;
	let frame = 0;

	function animate() {
		const now = performance.now();
		const elapsed = frame * frameMillis;//now - ZERO;
		for (let i = 0; i < LINE_GROUPS.length; i++) {
			styles[i] = createTransform(elapsed, LINE_GROUPS[i].fx, LINE_GROUPS[i].fy);
		}
		frame++;
		animationFrame = requestAnimationFrame(animate);
	}

	$effect(() => {
		console.debug('Prompt01 mounted', svgId);
		animationFrame = requestAnimationFrame(animate);
		return () => {
			console.debug('Prompt01 unmounted', svgId);
			if (animationFrame > 0) {
				cancelAnimationFrame(animationFrame);
			}
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
		<defs>
			{#each LINE_GROUPS as group}
				<mask id={group.mask.id}>
					{#if group.mask.type === 'rect'}
						<rect height={group.mask.r * 2} width={group.mask.r * 2} x={group.mask.cx - group.mask.r} y={group.mask.cy - group.mask.r}
									fill="#FFF" style={styles[group.index]} />
					{:else}
						<circle cx={group.mask.cx} cy={group.mask.cy} r={group.mask.r} fill="#FFF" style={styles[group.index]} />
					{/if}
				</mask>
			{/each}
		</defs>

		<rect id="background-rect" x="0" y="0" width={WIDTH} height={HEIGHT} fill="#FFF" />

		{#each LINE_GROUPS as group}
			<g mask="url(#{group.mask.id})">
				{#each group.lines as line}
					<line x1={line.x1} y1={line.y1} x2={line.x2} y2={line.y2} stroke="black" stroke-width={line.width} />
				{/each}
			</g>
		{/each}
	</svg>
</SVGContainer>