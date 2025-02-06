<script lang="ts">
	import SVGContainer from '$lib/SVGContainer.svelte';
	import { recorderState } from './ffmpegRecorder.svelte';

	const { svgId } = $props();

	type TransformFunction = (time: DOMHighResTimeStamp) => number;
	type Line = { x1: number; y1: number; x2: number; y2: number; width: number; };
	type Mask = { id: string; cx: number; cy: number; r: number; type: 'circle' | 'rect' | string; rHeight: number; };
	type LineGroup = {
		index: number;
		mask: Mask;
		lines: Line[];
		fx: TransformFunction;
		fy: TransformFunction;
		rotation: number;
	};

	const HEIGHT = 1080;
	const WIDTH = 1920;
	const INVERT = true;
	const R = Math.min(HEIGHT, WIDTH) * .7;

	let animationFrame = -1;
	let styles = $state(['']);
	let lineGroups = $state<LineGroup[]>([]);

	let totalY = 11+99; // line translation

	function getPingelingGroup(index: number): LineGroup {
		const left = index % 2 === 0;
		const horizontal = Math.random() > 0.5;
		const maskWidth = 500 * Math.random() + 50;
		const maskRatio = .5 + Math.random();
		return {
				index: index,
				mask: createMask(`mask-${index}`, left ? WIDTH/2 - HEIGHT *.8 : WIDTH/2 + HEIGHT/2, HEIGHT/2, maskWidth, 'rect', maskRatio),
				lines: generateLines(horizontal, 200 * Math.random() + 50, 150 * Math.random() + 10, horizontal ? -HEIGHT*3 : -WIDTH*3, horizontal ? HEIGHT*3 : WIDTH*3),
				fx: createAnimationFunction(Math.sin, 5000 + Math.random() * 2500, 100, Math.random() * 360),
				fy: createAnimationFunction(Math.cos, 1000 + Math.random() * 2500, HEIGHT/2 + maskWidth/2, Math.random() * 360),
				rotation: 1
			};
	}


	function generateFDMNGroups(): LineGroup[] {
		const groups: LineGroup[] = [];
		// for (let i = 0; i < 10; i++) {
		// 	groups.push(getPingelingGroup(i));
		// }

		groups.push({
			index: 0,
			mask: createMask(`mask-0`, WIDTH/2, HEIGHT + 450, 1960, 'rect', .5),
			lines: generateLines(true, 11, 99, -HEIGHT*3, HEIGHT*3),
			fx: createAnimationFunction(Math.sin, 5000, 0),
			fy: createAnimationFunction(Math.cos, 9000, HEIGHT/2),
			rotation: 0
		});

		return groups;
	}

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
				mask: createMask(`mask-${groups.length}`, maskCx, maskCy, r, Math.random() > 0.8 ? 'circle' : 'rect' ),
				lines: generateLines(horizontal, currentSpacing, currentWidth, start),
				fx: createAnimationFunction(Math.sin, timeRatio * phase, 250, phase),
				fy: createAnimationFunction(Math.cos, timeRatio, 5, phase),
				rotation: 0
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
		const end = stop < Number.MAX_VALUE ? stop : horizontal ? HEIGHT : WIDTH;
		for (let i = start; i < end; i += (spacing + width)) {
			lines.push({
				x1: horizontal ? -WIDTH : i,
				y1: horizontal ? i : -HEIGHT,
				x2: horizontal ? WIDTH * 2 : i,
				y2: horizontal ? i : HEIGHT * 2,
				width: width
			});
		}
		return lines;
	}

	function createMask(id: string, cx: number, cy: number, r: number, type: 'circle' | 'rect' = 'rect', rHeight = 1) {
		return { id, cx, cy, r, type, rHeight };
	}

	function createAnimationFunction(f = Math.sin, timeRatio = 1000, amplitude = R, phase = 0) {
		return (t: DOMHighResTimeStamp) => (f(t / timeRatio) * amplitude) + phase;
	}

	function createTransform(time: DOMHighResTimeStamp, fx: TransformFunction, fy: TransformFunction) {
		const cx = fx(time);
		const cy = fy(time);
		return `transform: translate(${cx}px, ${cy}px)`;
	}

	let frame = $state(0);
	let transY = $state(0);

	function animate() {
		const frameMillis = 1000 / recorderState.fps;
		let elapsed = frame * frameMillis;
		if (recorderState.recording) {
			elapsed = (recorderState.frame / (recorderState.fps)) * 1000;
			console.debug('Recording frame', recorderState.frame, elapsed);
		}
		setTimeout(()=> {
			for (let i = 0; i < lineGroups.length; i++) {
				styles[i] = createTransform(elapsed, lineGroups[i].fx, lineGroups[i].fy);
			}
			frame++;
			animationFrame = requestAnimationFrame(animate);
			transY = (transY + 1) % totalY;
		}, 0);
	}

	$effect(() => {
		console.debug('Prompt01 mounted', svgId);
		recorderState.width = WIDTH;
		recorderState.height = HEIGHT;
		recorderState.maxSeconds = 35;

		lineGroups = generateFDMNGroups();


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
			{#each lineGroups as group}
				<mask id={group.mask.id}>

					<g transform="rotate(-26) translate(0, 500)">
					{#if group.mask.type === 'rect'}
						<rect height={group.mask.r * 2 * group.mask.rHeight} width={group.mask.r * 2} x={group.mask.cx - group.mask.r} y={group.mask.cy - group.mask.r * group.mask.rHeight}
									fill="#FFF" style={styles[group.index]} />
					{:else}
						<circle cx={group.mask.cx} cy={group.mask.cy} r={group.mask.r} fill="#FFF" style={styles[group.index]} />
					{/if}
					</g>

				</mask>
			{/each}
		</defs>

		<rect id="background-rect" x="0" y="0" width={WIDTH} height={HEIGHT} fill={INVERT ? '#000' : '#FFF'} />

		{#each lineGroups as group}
			<g mask="url(#{group.mask.id})">
				<g transform="rotate({frame * group.rotation} {WIDTH / 2} {HEIGHT / 2})">
					{#each group.lines as line}
						<line x1={line.x1} y1={line.y1} x2={line.x2} y2={line.y2} stroke={INVERT ? 'white' : 'black'} stroke-width={line.width} transform={`translate(0,${transY})`}/>
					{/each}
				</g>
			</g>
		{/each}

		<circle cx={WIDTH / 2} cy={HEIGHT / 2} r={Math.min(WIDTH, HEIGHT) * .375} fill={INVERT ? '#000' : '#FFF'} stroke={INVERT ? '#000' : '#FFF'} stroke-width="33" />
	</svg>
</SVGContainer>