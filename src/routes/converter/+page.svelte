<script lang="ts">
	import { tweened } from 'svelte/motion';
	import { fade } from 'svelte/transition';
	import { confetti } from '@neoconfetti/svelte';
	import { FFmpeg } from '@ffmpeg/ffmpeg';
	import { onMount } from 'svelte';

	type Status = 'loading' | 'loaded' | 'convert.start' | 'convert.error' | 'convert.done';

	let status: Status = 'loading';
	let error = '';
	let ffmpeg: FFmpeg;
	let progress = tweened(0);

	async function readFile(file: File): Promise<Uint8Array> {
		return new Promise((resolve) => {
			const fileReader = new FileReader();

			fileReader.onload = () => {
				const { result } = fileReader;
				if (result instanceof ArrayBuffer) {
					resolve(new Uint8Array(result));
				}
			};

			fileReader.onerror = () => {
				error = 'Could not read file';
			};

			fileReader.readAsArrayBuffer(file);
		});
	}

	async function convertVideo(video: File) {
		status = 'convert.start';
		const videoData = await readFile(video);

		await ffmpeg.writeFile('input.webm', videoData);
		await ffmpeg.exec(['-i', 'input.webm', 'output.mp4']);
		const data = await ffmpeg.readFile('output.mp4');
		status = 'convert.done';
		return data as Uint8Array;
	}

	function downloadVideo(data: Uint8Array) {
		const a = document.createElement('a');
		a.href = URL.createObjectURL(new Blob([data.buffer], { type: 'video/mp4' }));
		a.download = 'video.mp4';

		setTimeout(() => {
			a.click();
		}, 1000);
	}

	async function handelDrop(e: DragEvent) {
		if (!e.dataTransfer) return;

		if (e.dataTransfer.files.length > 1) {
			error = 'Upload one file';
		}

		const [file] = e.dataTransfer.files;
		if (file?.type === 'video/webm') {
			error = '';
			const data = await convertVideo(file);
			downloadVideo(data);
		} else {
			error = 'Only WebM is supported';
		}
	}

	async function loadFFmpeg() {
		const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.4/dist/esm';

		ffmpeg = new FFmpeg();

		// ffmpeg.on('log', (e) => console.log(e));

		ffmpeg.on('progress', ({ progress, time }) => {
			console.log({ progress, time });

			$progress = progress * 100;
			// messageRef.current.innerHTML = `${progress * 100} % (transcoded time: ${time / 1000000} s)`;
		});

		ffmpeg.load({
			coreURL: `${baseURL}/ffmpeg-core.js`,
			wasmURL: `${baseURL}/ffmpeg-core.wasm`,
			workerURL: `${baseURL}/ffmpeg-core.worker.js`
		});

		status = 'loaded';
	}

	onMount(() => {
		loadFFmpeg();
	});
</script>

<h1>WebM to MP4 Converter</h1>
<div
	on:drop|preventDefault={handelDrop}
	on:dragover|preventDefault={handelDrop}
	data-status={status}
	role="region"
	aria-label="Drag and drop area"
>
	{#if status === 'loading'}
		<p in:fade>Loading FFmpeg...</p>
	{/if}

	{#if status === 'loaded'}
		<p in:fade>Drag and Drop</p>
	{/if}

	{#if status === 'convert.start'}
		<p in:fade>Converting video</p>
		<div>
			<div style:--progress="{$progress}%">{$progress.toFixed(0)}%</div>
		</div>
	{/if}

	{#if status === 'convert.done'}
		<div use:confetti />
		<p in:fade>Done 🎉</p>
	{/if}
</div>

<style>
	/* body {
		height: 100vh;
		width: 100vw;
	} */
</style>
