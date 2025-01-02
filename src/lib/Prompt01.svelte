<script lang="ts">
	const { svgId } = $props();

	const HEIGHT = 1080;
	const WIDTH = 1080;
	const ZERO = performance.now();

	const R = 250;
	let circle: SVGCircleElement | undefined = $state(undefined);
	let animationFrame = -1;
	const LINES = generateLines(10, 0, false);

	function generateLines(spacing = 10, start = 0, horizontal = true) {
		const lines = [];
		const end = horizontal ? WIDTH : HEIGHT;
		for (let i = start; i < end; i += spacing) {
			lines.push({
				x1: horizontal ? 0 : i,
				y1: horizontal ? i : 0,
				x2: horizontal ? WIDTH : i,
				y2: horizontal ? i : HEIGHT
			});
		}
		return lines;
	}

	function animate() {
		const now = performance.now();
		const elapsed = now - ZERO;
		cx = Math.sin(elapsed / 1000) * R + WIDTH / 2;
		animationFrame = requestAnimationFrame(animate);
	}

	$effect(() => {
		console.debug('Prompt01 mounted', svgId);
		if (circle) {
			animationFrame = requestAnimationFrame(animate);
		}
		return () => {
			console.debug('Prompt01 unmounted', svgId);
			if (animationFrame > 0) {
				cancelAnimationFrame(animationFrame);
			}
		};
	});


	let cx = $state(0);

	let mediaRecorder: MediaRecorder;
	let recordedChunks: BlobPart[] = [];
	let animationFrameId: number;

	function getSupportedMimeType(): string {
		const types = [
			'video/webm;codecs=h264',
			'video/webm;codecs=vp8',
			'video/webm',
			'video/mp4'
		];

		for (const type of types) {
			if (MediaRecorder.isTypeSupported(type)) {
				console.log('Using MIME type:', type);
				return type;
			}
		}
		throw new Error('No supported MIME types found for MediaRecorder');
	}

	async function startRecording() {
		const svgElement = document.getElementById(svgId);
		if (!svgElement) return;

		// Create a canvas element
		const canvas = document.createElement('canvas');
		const svgRect = svgElement.getBoundingClientRect();
		canvas.width = svgRect.width;
		canvas.height = svgRect.height;
		const ctx = canvas.getContext('2d');

		if (!ctx) return;

		// Get canvas stream
		const stream = canvas.captureStream(60); // Back to 60 FPS for smooth animation

		try {
			mediaRecorder = new MediaRecorder(stream, {
				mimeType: getSupportedMimeType(),
				videoBitsPerSecond: 2500000
			});

			mediaRecorder.ondataavailable = (event) => {
				if (event.data.size > 0) {
					recordedChunks.push(event.data);
				}
			};

			mediaRecorder.onerror = (event) => {
				console.error('MediaRecorder error:', event);
				stopRecording();
			};

			mediaRecorder.onstop = () => {
				cancelAnimationFrame(animationFrameId);
				const blob = new Blob(recordedChunks, {
					type: mediaRecorder.mimeType
				});

				const downloadUrl = URL.createObjectURL(blob);
				const a = document.createElement('a');
				a.href = downloadUrl;
				a.download = `animation.${mediaRecorder.mimeType.includes('mp4') ? 'mp4' : 'webm'}`;
				a.click();

				URL.revokeObjectURL(downloadUrl);
				recordedChunks = [];
			};

			// Start recording
			mediaRecorder.start(1000); // Collect data every second

			// Animation loop to capture SVG changes
			function drawFrame() {
				// Convert the current SVG state to a data URL
				const svgData = new XMLSerializer().serializeToString(svgElement);
				const svgBlob = new Blob([svgData], { type: 'image/svg+xml' });
				const url = URL.createObjectURL(svgBlob);

				const img = new Image();
				img.onload = () => {
					ctx.clearRect(0, 0, canvas.width, canvas.height);
					ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
					URL.revokeObjectURL(url);
				};
				img.src = url;

				animationFrameId = requestAnimationFrame(drawFrame);
			}

			drawFrame();

		} catch (error) {
			console.error('Error setting up MediaRecorder:', error);
			isRecording = false;
		}
	}

	function stopRecording() {
		if (mediaRecorder && mediaRecorder.state !== 'inactive') {
			mediaRecorder.stop();
		}
		if (animationFrameId) {
			cancelAnimationFrame(animationFrameId);
		}
	}

	let isRecording = $state(false);

	function toggleRecording() {
		if (!isRecording) {
			startRecording();
			isRecording = true;
		} else {
			stopRecording();
			isRecording = false;
		}
	}

	// Cleanup on component unmount
	$effect(() => {
		return () => {
			if (animationFrameId) {
				cancelAnimationFrame(animationFrameId);
			}
			if (mediaRecorder && mediaRecorder.state !== 'inactive') {
				mediaRecorder.stop();
			}
		};
	});
</script>

<button on:click={toggleRecording}>
	{isRecording ? 'Stop Recording' : 'Start Recording'}
</button>

<svg
	id={svgId}
	viewBox="0 0 {WIDTH} {HEIGHT}"
	xmlns="http://www.w3.org/2000/svg"
	width={`${WIDTH}`}
	height={`${HEIGHT}`}
>
	<defs>
		<mask id="circle01">
			<circle bind:this={circle} r={R} cx={cx} cy={HEIGHT / 2} stroke="black" stroke-width="0" fill="#FFF" />
		</mask>
	</defs>
	<rect x="0" y="0" width={WIDTH} height={HEIGHT} fill="#FFF" />
	<g id="lines">
		<line x1="0" y1="0" x2="1080" y2="1080" stroke="black" stroke-width="0" />
		<line x1="0" y1="1080" x2="1080" y2="0" stroke="black" stroke-width="0" />
	</g>

	<g id="line-group-01" mask="url(#circle01)">
		{#each LINES as line}
			<line x1={line.x1} y1={line.y1} x2={line.x2} y2={line.y2} stroke="black" stroke-width="1" />
		{/each}
	</g>
</svg>

<style>

</style>
