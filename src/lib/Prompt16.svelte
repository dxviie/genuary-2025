<script lang="ts">
	// eslint-disable-next-line @typescript-eslint/ban-ts-comment
	//@ts-expect-error
	import mixbox from 'mixbox';
	import SVGContainer from '$lib/SVGContainer.svelte';

	var rgb1 = 'rgb(0, 33, 133)';  // blue
	var rgb2 = 'rgb(252, 211, 0)'; // yellow
	var t = 0.5;                   // mixing ratio

	let mixed = $state(mixbox.lerp(rgb1, rgb2, 0));


	const { svgId } = $props();

	const WIDTH = 1080;
	const HEIGHT = 1080;

	let startTime;
	let animationId;


	$effect(() => {
		animationId = requestAnimationFrame(animate);
		return () => {
			cancelAnimationFrame(animationId);
		};
	});

	function animate(timestamp) {
		if (!startTime) startTime = timestamp;
		const elapsed = (timestamp - startTime) / 1000;
		const osc = (Math.sin(elapsed) + 1) / 2;
		mixed = mixbox.lerp(rgb1, rgb2, osc);
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
		<rect x={0} y={0} width={WIDTH} height={HEIGHT} fill={mixed}></rect>
	</svg>

</SVGContainer>