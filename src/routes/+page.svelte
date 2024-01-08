<script lang="ts">
	type Status = 'ready' | 'ready.countdown' | 'recording';
	type Optional<T> = T | undefined;

	export let width: Optional<number> = undefined;
	export let height: Optional<number> = undefined;
	export let audio = true;
	export let frameRate = 60;

	let stream: MediaStream;
	let recorder: MediaRecorder;
	let status: Status = 'ready';
	let videoChunks = [] as Blob[];
	let videoEl: HTMLVideoElement;
	let timerInterval: number;
	let timer = 3;
	const mimeType = 'video/webm;codecs=vp9';

	async function startTimer() {
		status = 'ready.countdown';

		return new Promise((resolve) => {
			timerInterval = setInterval(() => {
				timer--;
				if (timer === 0) {
					clearInterval(timerInterval);
					resolve(timer);
				}
			}, 1000);
		});
	}

	function downloadRecording(videoChunks: Blob[]) {
		const blob = new Blob(videoChunks, { type: mimeType });
		const a = document.createElement('a');
		a.href = URL.createObjectURL(blob);
		a.download = 'video.webm';
		a.click();
	}

	function stopRecording() {
		status = 'ready';
		stream.getTracks().forEach((track) => track.stop());
	}

	function startRecording() {
		status = 'recording';
		recorder = new MediaRecorder(stream, { mimeType });

		recorder.addEventListener('dataavailable', (e) => {
			videoChunks.push(e.data);
			videoEl.src = URL.createObjectURL(e.data);
		});

		recorder.addEventListener('stop', () => {
			downloadRecording(videoChunks);
			videoChunks = [];
			recorder && recorder.stop();
			clearInterval(timerInterval);
			timer = 3;
		});

		recorder.start();
	}

	async function prepareRecording() {
		try {
			stream = await navigator.mediaDevices.getDisplayMedia({
				video: { width, height, frameRate },
				audio
			});

			stream.addEventListener('inactive', stopRecording);

			await startTimer();
			startRecording();
		} catch (err) {
			console.error(`Error: ${err}`);
		}
	}

	function handelKeyDown(e: KeyboardEvent) {
		const isReady = status === 'ready';
		switch (e.key) {
			case 'R':
				isReady && prepareRecording();
			case 'S':
				!isReady && stopRecording();
		}
	}
</script>

<svelte:window on:keydown={handelKeyDown} />

<video bind:this={videoEl} controls />

{#if status.startsWith('ready')}
	<div class="recorder" data-state={status}>
		<button
			style="height: 3rem; width: 3rem; background-color: red; border: 4px solid white; border-radius: 100%; "
			on:click={prepareRecording}
			class="record"
		>
			<div class="circle">
				{#if status === 'ready.countdown'}
					{timer}
				{/if}
			</div>
		</button>

		<div class="info">
			{#if status === 'ready'}
				<p class="shortcut">Shift + R</p>
				<p class="description">To start recording</p>
			{/if}

			{#if status === 'ready.countdown'}
				<p class="shortcut">Shift + S</p>
				<p class="description">To stop recording</p>
			{/if}
		</div>
	</div>
{/if}

<style>
	[data-state='ready.countdown'] {
		&.circle {
		}
	}
</style>
