<script lang="ts">
	const { svgId } = $props();

	const HEIGHT = 1080;
	const WIDTH = 1080;
	const ZERO = performance.now();

	type TransformFunction = (time: DOMHighResTimeStamp) => number;
	type Line = { x1: number; y1: number; x2: number; y2: number; width: number; };
	type Mask = { id: string; cx: number; cy: number; r: number; };
	type LineGroup = {
		index: number;
		mask: Mask;
		lines: Line[];
		fx: TransformFunction;
		fy: TransformFunction;
		fr: TransformFunction;
	};

	let styles = $state(['']);
	const R = 250;
	let animationFrame = -1;
	const LINE_GROUPS = generateLineGroups();

	function generateLineGroups() {
		const minWidth = 0.5;
		const maxWidth = 5;
		const minSpacing = 5;
		const maxSpacing = 50;
		const horizontal = false;
		let currentWidth = Math.random() * (maxWidth - minWidth) + minWidth;
		let currentSpacing = Math.random() * (maxSpacing - minSpacing) + minSpacing;
		let availableSpace = currentSpacing;
		let start = 0;
		const groups: LineGroup[] = [];
		while (availableSpace > 0) {
			groups.push({
				index: groups.length,
				mask: createMask(`mask-${groups.length}`, WIDTH / 2, HEIGHT / 2, R),
				lines: generateLines(horizontal, currentSpacing, currentWidth, start),
				fx: createAnimationFunction(Math.sin, Math.random() * 2500 + 1000, 250),
				fy: createAnimationFunction(Math.cos, Math.random() * 2500 + 1000, 50),
				fr: createAnimationFunction(() => 1, 1)
			});
			availableSpace -= currentWidth;
			start += currentWidth;

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
		return { id, cx, cy, r };
	}

	function createAnimationFunction(f = Math.sin, timeRatio = 1000, amplitude = R) {
		return (t: DOMHighResTimeStamp) => f(t / timeRatio) * amplitude;
	}

	function createTransform(time: DOMHighResTimeStamp, fx: TransformFunction, fy: TransformFunction, fr: TransformFunction) {
		const cx = fx(time);
		const cy = fy(time);
		const r = fr(time);
		return `transform: translate(${cx}px, ${cy}px)`;
	}

	function animate() {
		const now = performance.now();
		const elapsed = now - ZERO;
		for (let i = 0; i < LINE_GROUPS.length; i++) {
			styles[i] = createTransform(elapsed, LINE_GROUPS[i].fx, LINE_GROUPS[i].fy, LINE_GROUPS[i].fr);
		}
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
				<circle r={group.mask.r} cx={group.mask.cx} cy={group.mask.cy} stroke="black" stroke-width="0" fill="#FFF"
								style={styles[group.index]} />
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