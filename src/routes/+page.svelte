<script lang="ts">
	import { onDestroy } from 'svelte';

	const NUM_STEPS = 16;
	const STEPS_PER_BEAT = 4;
	const NUM_PITCHES = 12;
	const BASE_MIDI_NOTE = 60; // C4

	interface Step {
		pitches: boolean[];
	}

	interface Notes {
		steps: Step[];
	}

	function notesEmpty(): Notes {
		return {
			steps: Array.from({ length: NUM_STEPS }, () => ({
				pitches: Array(NUM_PITCHES).fill(false)
			}))
		};
	}

	let notes: Notes = notesEmpty();
	let isPlaying = false;
	let currentStep = 0;
	let bpm = 120;

	let timeoutId: number | null = null; // browser internal for the step scheduler
	let _audioCtxCache: AudioContext | null = null; // browser internal

	const noteNames = ['C4', 'C#4', 'D4', 'D#4', 'E4', 'F4', 'F#4', 'G4', 'G#4', 'A4', 'A#4', 'B4'];

	function midiToFreq(midi: number) {
		return 440 * 2 ** ((midi - 69) / 12);
	}

	function ensureAudioContext(): AudioContext {
		if (_audioCtxCache === null || _audioCtxCache.state === 'closed') {
			_audioCtxCache = new window.AudioContext();
		}
		return _audioCtxCache;
	}

	function playNote(midiNote: number, time: number, duration = 0.25) {
		const ctx = ensureAudioContext();
		const osc = ctx.createOscillator();
		const gain = ctx.createGain();

		osc.type = 'square';
		osc.frequency.value = midiToFreq(midiNote);

		gain.gain.setValueAtTime(0, time); // starting at 0 prevents clicks
		gain.gain.linearRampToValueAtTime(0.3, time + 0.01);
		gain.gain.linearRampToValueAtTime(0.0, time + duration); // fade out

		osc.connect(gain);
		gain.connect(ctx.destination);

		osc.start(time);
		osc.stop(time + duration + 0.05);
	}

	function toggleCell(step: number, pitchIndex: number) {
		notes.steps[step].pitches[pitchIndex] = !notes.steps[step].pitches[pitchIndex];
		notes = notes;
	}

	function stepDurationSeconds() {
		return 60 / (bpm * STEPS_PER_BEAT);
	}

	function scheduleStep(step: number) {
		const ctx = ensureAudioContext();
		const time = ctx.currentTime;
		const pitches = notes.steps[step].pitches;
		for (const [pitchIndex, active] of pitches.entries()) {
			if (active) {
				const midi = BASE_MIDI_NOTE + (NUM_PITCHES - 1 - pitchIndex);
				playNote(midi, time, stepDurationSeconds() * 0.9);
			}
		}
	}

	function scheduleNextStep() {
		if (!isPlaying) return;
		currentStep = (currentStep + 1) % NUM_STEPS;
		scheduleStep(currentStep);
		const durationMs = stepDurationSeconds() * 1000;
		timeoutId = window.setTimeout(scheduleNextStep, durationMs);
	}

	async function start() {
		if (isPlaying) return;
		isPlaying = true;

		const ctx = ensureAudioContext();
		if (ctx.state === 'suspended') {
			// NOTE: `ctx.resume()` must be called from a user gesture, so we don't do this in `ensureAudioContext()`
			await ctx.resume();
		}

		currentStep = 0;
		scheduleStep(currentStep);
		const durationMs = stepDurationSeconds() * 1000;
		timeoutId = window.setTimeout(scheduleNextStep, durationMs);
	}

	function stop() {
		if (!isPlaying) return;
		isPlaying = false;
		if (timeoutId !== null) {
			clearTimeout(timeoutId);
			timeoutId = null;
		}
	}

	function clearPattern() {
		notes = notesEmpty();
	}

	onDestroy(() => {
		if (timeoutId !== null) {
			clearTimeout(timeoutId);
		}
	});
</script>

<main class="flex min-h-screen items-center justify-center bg-slate-950 px-4 py-10 text-slate-100">
	<div class="w-full max-w-5xl space-y-6">
		<header class="space-y-1">
			<h1 class="text-3xl font-semibold tracking-tight">MIDI Grid</h1>
			<p class="text-sm text-slate-400">
				Click cells to toggle notes, then press play to hear the loop.
			</p>
		</header>

		<section class="flex flex-wrap items-center gap-4">
			<button
				type="button"
				on:click={isPlaying ? stop : start}
				class="inline-flex items-center gap-2 rounded-md px-4 py-2 text-sm font-medium
					{isPlaying
					? 'bg-rose-500 text-white hover:bg-rose-400'
					: 'bg-emerald-500 text-white hover:bg-emerald-400'}"
			>
				<span>{isPlaying ? 'Stop' : 'Play'}</span>
			</button>

			<div class="flex items-center gap-2 text-sm">
				<label for="bpm" class="text-slate-300">BPM</label>
				<input
					id="bpm"
					type="number"
					min="40"
					max="240"
					bind:value={bpm}
					class="w-20 rounded-md border border-slate-700 bg-slate-900 px-2 py-1 text-right text-sm
						focus-visible:ring-2 focus-visible:ring-emerald-500/70 focus-visible:outline-none"
				/>
			</div>

			<div class="flex items-center gap-2 text-xs text-slate-400">
				<button
					type="button"
					on:click={clearPattern}
					class="rounded-md border border-slate-700 px-2 py-1 hover:bg-slate-800"
				>
					Clear
				</button>
			</div>
		</section>

		<section
			class="flex gap-2 overflow-x-auto rounded-xl border border-slate-800 bg-slate-900/60 p-3 shadow-lg shadow-black/40"
		>
			<div class="flex flex-col justify-between py-6 pr-2 text-xs text-slate-400">
				{#each [...Array(NUM_PITCHES).keys()] as i (i)}
					<div class="flex h-8 items-center justify-end pr-1">
						<span>{noteNames[NUM_PITCHES - 1 - i]}</span>
					</div>
				{/each}
			</div>

			<div class="relative flex-1">
				<div class="grid auto-cols-[minmax(1.5rem,2.5rem)] grid-flow-col gap-0.5">
					{#each notes.steps as column, stepIndex (stepIndex)}
						<div class="flex flex-col gap-0.5">
							{#each column.pitches as active, pitchIndex (pitchIndex)}
								<button
									type="button"
									on:click={() => toggleCell(stepIndex, pitchIndex)}
									class={`h-8 w-8 rounded-sm border text-xs transition
										${
											active
												? 'border-emerald-400 bg-emerald-500/70 shadow-[0_0_10px_rgba(16,185,129,0.7)]'
												: 'border-slate-800 bg-slate-900/70 hover:border-slate-600 hover:bg-slate-800'
										}
										${
											isPlaying && currentStep === stepIndex
												? 'ring-2 ring-emerald-400/80 ring-offset-2 ring-offset-slate-900'
												: ''
										}
									`}
									aria-label="Toggle note"
								></button>
							{/each}
						</div>
					{/each}
				</div>
			</div>
		</section>
	</div>
</main>
