<script lang="ts">
	const { svgId } = $props();

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
		if (!svgElement) {
			console.error('No SVG element found (looking for id "prompt-svg")');
			return;
		}

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
		finalCanvas.width = 297 * 4;
		finalCanvas.height = 210 * 4;
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

<div class="buttons">
	<button onclick={() => window.location.reload()}>Roll the Dice</button>
	<button onclick={toggleRecording}>
		{isRecording ? 'Stop Recording' : 'Start Recording'}
	</button>
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