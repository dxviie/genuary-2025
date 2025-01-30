<script lang="ts">
	import { FFmpeg } from '@ffmpeg/ffmpeg';
	import { recorderState } from '$lib/ffmpegRecorder.svelte';

	const { svgId } = $props();

	const isLocalhost = import.meta.env.DEV; // true in development


	let mediaRecorder: MediaRecorder;
	let recordedChunks: BlobPart[] = [];
	let animationFrameId: number;

	function getSupportedMimeType(): string {
		// Prioritize H.264 codec since that matches your FFmpeg command
		const types = [
			'video/mp4;codecs=h264',
			'video/webm;codecs=h264',
			'video/webm;codecs=vp9', // Fallback to VP9 which can provide similar quality
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
		if (!svgElement) {
			console.error('No SVG element found (looking for id "prompt-svg")');
			return;
		}

		// Create a canvas element
		const canvas = document.createElement('canvas');
		canvas.width = 1080;//svgRect.width;
		canvas.height = 1080;//svgRect.height;
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
				a.download = `genuary25.d17e.dev-${createDateTimeTimestamp()}.${mediaRecorder.mimeType.includes('mp4') ? 'mp4' : 'webm'}`;
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

	function createDateTimeTimestamp() {
		const now = new Date();
		const year = now.getFullYear();
		const month = String(now.getMonth() + 1).padStart(2, '0');
		const day = String(now.getDate()).padStart(2, '0');
		const hours = String(now.getHours()).padStart(2, '0');
		const minutes = String(now.getMinutes()).padStart(2, '0');
		const seconds = String(now.getSeconds()).padStart(2, '0');
		return `${year}-${month}-${day}_${hours}-${minutes}-${seconds}`;
	}

	function saveSvgAsPng() {
		const svgElement = document.getElementById(svgId);
		if (!svgElement) {
			console.error('No SVG element found (looking for id "prompt-svg")');
			return;
		}

		const svgData = new XMLSerializer().serializeToString(svgElement);
		const finalCanvas = document.createElement('canvas');
		finalCanvas.width = svgElement.clientWidth;
		finalCanvas.height = svgElement.clientHeight;
		const ctx = finalCanvas.getContext('2d');
		// First draw the SVG
		const img = new Image();
		const svgBlob = new Blob([svgData], { type: 'image/svg+xml;charset=utf-8' });
		const url = URL.createObjectURL(svgBlob);

		img.onload = () => {
			if (!ctx) {
				console.warn('no CTX');
				return;
			}
			ctx.drawImage(img, 0, 0, finalCanvas.width, finalCanvas.height);
			// Create download link
			const pngUrl = finalCanvas.toDataURL('image/png');
			const link = document.createElement('a');
			link.href = pngUrl;
			link.download = `genuary25.d17e.dev-${createDateTimeTimestamp()}.png`;
			document.body.appendChild(link);
			link.click();
			document.body.removeChild(link);
			URL.revokeObjectURL(url);
		};
		img.src = url;
	}

	// ffmpeg stuff
	let ffmpeg;
	let frames = [];

	// Cleanup on component unmount
	$effect(() => {
		if (animationFrameId) {
			cancelAnimationFrame(animationFrameId);
		}
		if (mediaRecorder && mediaRecorder.state !== 'inactive') {
			mediaRecorder.stop();
		}
		if (isLocalhost) {
			loadFfmpeg();
		}
	});
	let loaded = $state(false);


	const loadFfmpeg = async () => {
		ffmpeg = new FFmpeg();
		ffmpeg.on('log', ({ message }) => {
			console.log(message);
		});
		// toBlobURL is used to bypass CORS issue, urls with the same
		// domain can be used directly.
		await ffmpeg.load({
			coreURL: `/ffmpeg/ffmpeg-core.js`,
			wasmURL: `/ffmpeg/ffmpeg-core.wasm`
		});
		loaded = true;
		console.log('FFmpeg loaded');
	};

	async function toggleRecordingFfmpeg() {
		if (!recorderState.recording) {
			startRecordingFfmpeg();
		} else {
			stopRecordingFfmpeg();
		}
	}

	async function startRecordingFfmpeg() {
		frames = [];
		recorderState.recording = true;
		recorderState.frame = 0;
		recordFrame();
	}

	async function stopRecordingFfmpeg() {
		recorderState.recording = false;
		await generateVideo();
	}

	async function recordFrame() {
		if (!recorderState.recording) return;

		const svgElement = document.getElementById(svgId);
		if (!svgElement) {
			console.error('No SVG element found (looking for id "prompt-svg")');
			return;
		}

		// Convert SVG to canvas
		const svgData = new XMLSerializer().serializeToString(svgElement);
		const svgBlob = new Blob([svgData], { type: 'image/svg+xml' });
		const svgUrl = URL.createObjectURL(svgBlob);

		// Create a canvas element
		const frameCanvas = document.createElement('canvas');
		frameCanvas.width = recorderState.width;
		frameCanvas.height = recorderState.height;
		const ctx = frameCanvas.getContext('2d');

		const img = new Image();
		img.onload = () => {
			// const ctx = frameCanvas.getContext('2d');
			ctx.drawImage(img, 0, 0);

			// Store frame
			frames.push(frameCanvas.toDataURL('image/png'));
			recorderState.frame += 1;
			if (recorderState.frame >= recorderState.fps * recorderState.maxSeconds) {
				stopRecordingFfmpeg();
			}

			// Schedule next frame
			if (recorderState.recording) {
				requestAnimationFrame(recordFrame);
			}

			URL.revokeObjectURL(svgUrl);
		};
		img.src = svgUrl;
	}

	async function generateVideo() {
		// Write frames to FFmpeg
		for (let i = 0; i < frames.length; i++) {
			const base64Data = frames[i].split(',')[1];
			const binaryData = atob(base64Data);
			const data = new Uint8Array(binaryData.length);
			for (let j = 0; j < binaryData.length; j++) {
				data[j] = binaryData.charCodeAt(j);
			}
			await ffmpeg.writeFile(`frame${i.toString().padStart(4, '0')}.png`, data);
		}
		console.log('========== All frames written');

		// Generate video from frames
		await ffmpeg.exec([
			'-framerate', `${recorderState.fps}`,
			'-pattern_type', 'glob',
			'-i', 'frame*.png',
			'-c:v', 'libx264',
			'-pix_fmt', 'yuv420p',
			'-crf', '23',
			'output.mp4'
		]);
		console.log('========== Video generated');

		// Read the output file
		const data = await ffmpeg.readFile('output.mp4');
		console.log('========== Video data:', data);
		const blob = new Blob([data], { type: 'video/mp4' });
		const url = URL.createObjectURL(blob);

		// Create download link
		const a = document.createElement('a');
		a.href = url;
		a.download = `genuary25.d17e.dev-${createDateTimeTimestamp()}.mp4`;
		console.log('========== Downloading video');
		a.click();

		// Cleanup
		URL.revokeObjectURL(url);
		frames = [];
	}
</script>

<div class="buttons">
	<button onclick={() => window.location.reload()}>Roll the Dice</button>
	<button onclick={toggleRecording}>
		{isRecording ? 'Stop Recording' : 'Start Recording'}
	</button>
	{#if isLocalhost}
		<button onclick={toggleRecordingFfmpeg}>
			{recorderState.recording ? 'Stop Recording FFMPEG' : 'Start Recording FFMPEG'}
		</button>
	{/if}
	<button onclick={saveSvgAsPng}>
		Save as PNG
	</button>
</div>

<style>
    :global(body) {
        font-family: "Courier New", serif;
        line-height: 1.4rem;
    }

    :global(a) {
        color: #000;
        text-decoration: none;
        border-bottom-style: dashed;
        border-bottom-width: 1px;
    }

    :global(a:hover) {
        color: darkorange;
    }

    :global(main) {
        display: flex;
        flex-direction: column;
        align-items: center;
        padding: 1rem .5rem;
    }

    :global(h1) {
        text-align: center;
    }

    @media (max-width: 600px) {
        :global(h1) {
            font-size: 1.5rem;
        }
    }

    :global(button) {
        padding: 0.5rem 1rem;
        font-size: 1rem;
        font-family: "Courier New", serif;
        font-weight: bold;
        background-color: #fff;
        color: #000;
        border-radius: 1rem;
        cursor: pointer;
        border-style: dashed;
        border-color: #000;
        border-width: 1px;
    }

    :global(button:active) {
        transform: translate(0, 1px);
    }

    :global(svg) {
        width: 100%;
        height: auto;
        object-fit: contain;
    }

    .buttons {
        display: flex;
        flex-direction: row;
        justify-content: center;
        gap: 1rem;
    }
</style>