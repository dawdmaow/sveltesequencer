<script lang="ts">
	import { onDestroy } from 'svelte';

	const MAX_STEPS = 32;
	const NUM_OCTAVES = 6;
	const NUM_PITCHES = NUM_OCTAVES * 12;
	const BASE_MIDI_NOTE = 48; // C3
	const START_OCTAVE = 1;
	const MIN_BPM = 40;
	const MAX_BPM = 300;
	const MIN_BEATS_PER_MEASURE = 1;
	const MAX_BEATS_PER_MEASURE = 16;
	const MIN_STEPS_PER_BEAT = 1;
	const MAX_STEPS_PER_BEAT = 16;
	const MIN_MEASURES = 1;
	const MAX_MEASURES = 8;
	const MAX_PATTERNS = 8;
	const PATTERN_IDS = ['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H'];

	interface Step {
		pitches: boolean[];
	}

	interface Pattern {
		id: string;
		steps: Step[];
	}

	function createPattern(id: string, beats: number, steps: number): Pattern {
		const clampedBeats = clampBeatsPerMeasure(beats);
		const clampedSteps = clampStepsPerBeat(steps);
		const stepsCount = clampedBeats * clampedSteps;
		return {
			id,
			steps: Array.from({ length: stepsCount }, () => ({
				pitches: Array(NUM_PITCHES).fill(false)
			}))
		};
	}

	type Scale =
		| 'Major'
		| 'Natural Minor'
		| 'Harmonic Minor'
		| 'Melodic Minor'
		| 'Dorian'
		| 'Phrygian'
		| 'Lydian'
		| 'Mixolydian'
		| 'Locrian'
		| 'Pentatonic Major'
		| 'Pentatonic Minor'
		| 'Blues'
		| 'Chromatic';

	let patterns = $state<Pattern[]>([]);
	let patternIdSequence = $state<string[]>([]);
	let selectedSequenceIndex = $state(0);
	let isPlaying = $state(false);
	let currentStep = $state(0); // TODO: this should be local; need to store current sequence index as well (and give it to playStep())
	let bpm = $state(120);
	let beatsPerMeasure = $state(4);
	let stepsPerBeat = $state(4);
	let numMeasures = $state(1);
	let key = $state('C');
	let scale = $state<Scale>('Natural Minor');
	let allowNonScaleNotes = $state(true);
	let isPainting = $state(false);
	let paintTargetState = $state(true);

	let selectedPatternId = $derived(patternIdSequence[selectedSequenceIndex] ?? PATTERN_IDS[0]);

	let displayedPatternId = $derived(
		isPlaying
			? (patternIdSequence[Math.floor(currentStep / stepsPerPattern())] ?? PATTERN_IDS[0])
			: selectedPatternId
	);

	function reset() {
		patterns = PATTERN_IDS.map((id) => createPattern(id, 4, 4));
		patternIdSequence = ['A'];
		selectedSequenceIndex = 0;
		key = 'C';
		scale = 'Natural Minor';
		allowNonScaleNotes = true;
		bpm = 120;
		beatsPerMeasure = 4;
		stepsPerBeat = 4;
		numMeasures = 1;
	}
	reset();

	let timeoutId: number | null = null; // browser internal for the step scheduler
	let _audioCtxCache: AudioContext | null = null; // browser internal

	const keys = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'];
	const chromatic = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'];

	const scales: Record<Scale, number[]> = {
		Major: [0, 2, 4, 5, 7, 9, 11],
		'Natural Minor': [0, 2, 3, 5, 7, 8, 10],
		'Harmonic Minor': [0, 2, 3, 5, 7, 8, 11],
		'Melodic Minor': [0, 2, 3, 5, 7, 9, 11],
		Dorian: [0, 2, 3, 5, 7, 9, 10],
		Phrygian: [0, 1, 3, 5, 7, 8, 10],
		Lydian: [0, 2, 4, 6, 7, 9, 11],
		Mixolydian: [0, 2, 4, 5, 7, 9, 10],
		Locrian: [0, 1, 3, 5, 6, 8, 10],
		'Pentatonic Major': [0, 2, 4, 7, 9],
		'Pentatonic Minor': [0, 3, 5, 7, 10],
		Blues: [0, 3, 5, 6, 7, 10],
		Chromatic: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11]
	};

	function getChromaticIndex(noteName: string): number {
		return chromatic.indexOf(noteName);
	}

	function isRootNote(pitchIndex: number): boolean {
		const noteInOctave = (NUM_PITCHES - 1 - pitchIndex) % 12;
		const rootIndex = getChromaticIndex(key);
		return noteInOctave === rootIndex;
	}

	function isFifthNote(pitchIndex: number): boolean {
		const noteInOctave = (NUM_PITCHES - 1 - pitchIndex) % 12;
		const rootIndex = getChromaticIndex(key);
		const fifthIndex = (rootIndex + 7) % 12;
		return noteInOctave === fifthIndex;
	}

	function isInScale(pitchIndex: number): boolean {
		const noteInOctave = (NUM_PITCHES - 1 - pitchIndex) % 12;
		const rootIndex = getChromaticIndex(key);
		const scaleIntervals = scales[scale];
		const relativeInterval = (noteInOctave - rootIndex + 12) % 12;
		return scaleIntervals.includes(relativeInterval);
	}

	function isFifthInScale(pitchIndex: number): boolean {
		return isFifthNote(pitchIndex) && isInScale(pitchIndex);
	}

	function getCellClasses(pitchIndex: number, isActive: boolean): string {
		if (isRootNote(pitchIndex)) {
			return isActive
				? 'border-blue-400 bg-blue-500/70 shadow-lg shadow-blue-500/50'
				: 'border-blue-800/50 bg-blue-900/30 hover:border-blue-700 hover:bg-blue-900/50';
		}
		if (isFifthInScale(pitchIndex)) {
			return isActive
				? 'border-purple-400 bg-purple-500/70 shadow-lg shadow-purple-500/50'
				: 'border-purple-800/50 bg-purple-900/30 hover:border-purple-700 hover:bg-purple-900/50';
		}
		if (isFifthNote(pitchIndex)) {
			return isActive
				? 'border-purple-700/50 bg-purple-800/40 shadow-lg shadow-purple-900/30'
				: 'border-purple-900/30 bg-purple-950/20 hover:border-purple-800/40 hover:bg-purple-950/30';
		}
		if (isInScale(pitchIndex)) {
			return isActive
				? 'border-emerald-400 bg-emerald-500/70 shadow-lg shadow-emerald-500/50'
				: 'border-slate-700/50 bg-slate-800/40 hover:border-slate-600 hover:bg-slate-800/60';
		}
		return isActive
			? 'border-amber-400 bg-amber-500/50 shadow-lg shadow-amber-500/30'
			: 'border-slate-800/30 bg-slate-900/20 hover:border-slate-700/50 hover:bg-slate-800/30';
	}

	function getNoteLabelClasses(pitchIndex: number): string {
		if (isRootNote(pitchIndex)) {
			return 'font-semibold text-blue-400';
		}
		if (isFifthInScale(pitchIndex)) {
			return 'font-semibold text-purple-400';
		}
		if (isFifthNote(pitchIndex)) {
			return 'text-purple-600';
		}
		if (isInScale(pitchIndex)) {
			return 'text-slate-300';
		}
		return 'text-slate-500';
	}

	function getNoteName(pitchIndex: number): string {
		const octave = START_OCTAVE + Math.floor((NUM_PITCHES - 1 - pitchIndex) / 12);
		const note = chromatic[(NUM_PITCHES - 1 - pitchIndex) % 12];
		return `${note}${octave}`;
	}

	function clampBpm(value: number): number {
		if (!Number.isFinite(value)) return MIN_BPM;
		if (value < MIN_BPM) return MIN_BPM;
		if (value > MAX_BPM) return MAX_BPM;
		return value;
	}

	function clampBeatsPerMeasure(value: number): number {
		if (!Number.isFinite(value)) return MIN_BEATS_PER_MEASURE;
		if (value < MIN_BEATS_PER_MEASURE) return MIN_BEATS_PER_MEASURE;
		if (value > MAX_BEATS_PER_MEASURE) return MAX_BEATS_PER_MEASURE;
		return Math.trunc(value);
	}

	function clampStepsPerBeat(value: number): number {
		if (!Number.isFinite(value)) return MIN_STEPS_PER_BEAT;
		if (value < MIN_STEPS_PER_BEAT) return MIN_STEPS_PER_BEAT;
		if (value > MAX_STEPS_PER_BEAT) return MAX_STEPS_PER_BEAT;
		return Math.trunc(value);
	}

	function stepsPerPattern(): number {
		const beats = clampBeatsPerMeasure(beatsPerMeasure);
		const steps = clampStepsPerBeat(stepsPerBeat);
		return beats * steps;
	}

	function getPatternColor(id: string): string {
		const colors = [
			'bg-blue-600',
			'bg-emerald-600',
			'bg-purple-600',
			'bg-amber-600',
			'bg-rose-600',
			'bg-cyan-600',
			'bg-pink-600',
			'bg-indigo-600'
		];
		const index = PATTERN_IDS.indexOf(id);
		return index >= 0 && index < colors.length ? colors[index] : 'bg-slate-600';
	}

	function cyclePatternAtPosition(seqIndex: number, direction: 1 | -1) {
		if (seqIndex >= patternIdSequence.length) return;
		const currentId = patternIdSequence[seqIndex];
		const currentIndex = PATTERN_IDS.indexOf(currentId);
		const nextIndex = (currentIndex + direction + PATTERN_IDS.length) % PATTERN_IDS.length;
		const nextId = PATTERN_IDS[nextIndex];
		patternIdSequence[seqIndex] = nextId;
	}

	function totalSteps(): number {
		const stepsPerPat = stepsPerPattern();
		return patternIdSequence.length * stepsPerPat;
	}

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

	function setCell(step: number, pitchIndex: number, state: boolean) {
		if (!allowNonScaleNotes && !isInScale(pitchIndex)) return;
		const pattern = patterns.find((p) => p.id === displayedPatternId);
		if (!pattern || step >= pattern.steps.length) return;
		pattern.steps[step].pitches[pitchIndex] = state;
	}

	function stepDurationSeconds() {
		const numericBpm = clampBpm(Number(bpm));
		const steps = clampStepsPerBeat(stepsPerBeat);
		return 60 / (numericBpm * steps);
	}

	function playStep(step: number) {
		const ctx = ensureAudioContext();
		const time = ctx.currentTime;
		const stepsPerPat = stepsPerPattern();
		const sequenceIndex = Math.floor(step / stepsPerPat);
		const stepInPattern = step % stepsPerPat;

		if (sequenceIndex >= patternIdSequence.length) {
			return;
		}

		const patternId = patternIdSequence[sequenceIndex];

		const pattern = patterns.find((p) => p.id === patternId);
		if (!pattern || stepInPattern >= pattern.steps.length) {
			return;
		}

		const pitches = pattern.steps[stepInPattern].pitches;
		for (const [pitchIndex, active] of pitches.entries()) {
			if (active) {
				const midi = BASE_MIDI_NOTE + (NUM_PITCHES - 1 - pitchIndex);
				playNote(midi, time, stepDurationSeconds() * 0.9);
			}
		}
	}

	function startPlayingStepsUsingTimeout() {
		if (!isPlaying) return;
		const stepsInLoop = totalSteps();
		currentStep = (currentStep + 1) % stepsInLoop;
		playStep(currentStep);
		const durationMs = stepDurationSeconds() * 1000;
		timeoutId = window.setTimeout(startPlayingStepsUsingTimeout, durationMs);
	}

	async function start() {
		if (isPlaying) {
			return;
		}
		isPlaying = true;

		const ctx = ensureAudioContext();
		if (ctx.state === 'suspended') {
			// NOTE: `ctx.resume()` must be called from a user gesture, so we don't do this in `ensureAudioContext()`
			await ctx.resume();
		}

		currentStep = selectedSequenceIndex * stepsPerPattern();
		playStep(currentStep);
		const durationMs = stepDurationSeconds() * 1000;
		timeoutId = window.setTimeout(startPlayingStepsUsingTimeout, durationMs);
	}

	function stop() {
		if (!isPlaying) {
			return;
		}
		isPlaying = false;
		if (timeoutId !== null) {
			clearTimeout(timeoutId);
			timeoutId = null;
		}
	}

	function clearPattern() {
		const patternIndex = patterns.findIndex((p) => p.id === displayedPatternId);
		const pattern = patterns[patternIndex];
		if (!pattern) return;

		const clearedPattern = createPattern(displayedPatternId, beatsPerMeasure, stepsPerBeat);
		patterns[patternIndex] = clearedPattern;
	}

	$effect(() => {
		const stepsPerPat = stepsPerPattern();
		const needsUpdate = patterns.some((pattern) => pattern.steps.length !== stepsPerPat);
		if (!needsUpdate) return;

		for (const pattern of patterns) {
			const currentSteps = pattern.steps.length;
			if (currentSteps === stepsPerPat) continue;

			if (stepsPerPat > currentSteps) {
				for (let i = currentSteps; i < stepsPerPat; i++) {
					pattern.steps.push({
						pitches: Array(NUM_PITCHES).fill(false)
					});
				}
			} else {
				pattern.steps.length = stepsPerPat;
			}
		}
	});

	onDestroy(() => {
		if (timeoutId !== null) {
			clearTimeout(timeoutId);
		}
	});
</script>

<svelte:window
	on:keydown={(e) => {
		if (e.key !== ' ') return;
		if (e.target instanceof HTMLInputElement || e.target instanceof HTMLTextAreaElement) return;
		e.preventDefault();
		if (isPlaying) stop();
		else start();
	}}
	on:mouseup={() => {
		isPainting = false;
	}}
/>

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
					min={MIN_BPM}
					max={MAX_BPM}
					bind:value={bpm}
					on:blur={() => {
						bpm = clampBpm(bpm);
					}}
					on:keydown={(event) => {
						if (event.key === 'Enter') {
							bpm = clampBpm(bpm);
						}
					}}
					class="w-20 rounded-md border border-slate-700 bg-slate-900 px-2 py-1 text-right text-sm
						focus-visible:ring-2 focus-visible:ring-emerald-500/70 focus-visible:outline-none"
				/>
			</div>

			<div class="flex items-center gap-2 text-sm">
				<label for="beats" class="text-slate-300">Beats / bar</label>
				<input
					id="beats"
					type="number"
					min={MIN_BEATS_PER_MEASURE}
					max={MAX_BEATS_PER_MEASURE}
					bind:value={beatsPerMeasure}
					on:blur={() => {
						beatsPerMeasure = clampBeatsPerMeasure(beatsPerMeasure);
					}}
					on:keydown={(event) => {
						if (event.key === 'Enter') {
							beatsPerMeasure = clampBeatsPerMeasure(beatsPerMeasure);
						}
					}}
					class="w-20 rounded-md border border-slate-700 bg-slate-900 px-2 py-1 text-right text-sm
						focus-visible:ring-2 focus-visible:ring-emerald-500/70 focus-visible:outline-none"
				/>
			</div>

			<div class="flex items-center gap-2 text-sm">
				<label for="steps" class="text-slate-300">Steps / beat</label>
				<input
					id="steps"
					type="number"
					min={MIN_STEPS_PER_BEAT}
					max={MAX_STEPS_PER_BEAT}
					bind:value={stepsPerBeat}
					on:blur={() => {
						stepsPerBeat = clampStepsPerBeat(stepsPerBeat);
					}}
					on:keydown={(event) => {
						if (event.key === 'Enter') {
							stepsPerBeat = clampStepsPerBeat(stepsPerBeat);
						}
					}}
					class="w-24 rounded-md border border-slate-700 bg-slate-900 px-2 py-1 text-right text-sm
						focus-visible:ring-2 focus-visible:ring-emerald-500/70 focus-visible:outline-none"
				/>
			</div>

			<div class="flex items-center gap-2 text-xs text-slate-400">
				<label for="key" class="text-slate-300">Key</label>
				<select
					id="key"
					bind:value={key}
					class="rounded-md border border-slate-700 bg-slate-900 px-2 py-1 text-sm
						focus-visible:ring-2 focus-visible:ring-emerald-500/70 focus-visible:outline-none"
				>
					{#each keys as k (k)}
						<option value={k}>{k}</option>
					{/each}
				</select>
			</div>

			<div class="flex items-center gap-2 text-xs text-slate-400">
				<label for="scale" class="text-slate-300">Scale</label>
				<select
					id="scale"
					bind:value={scale}
					class="rounded-md border border-slate-700 bg-slate-900 px-2 py-1 text-sm
						focus-visible:ring-2 focus-visible:ring-emerald-500/70 focus-visible:outline-none"
				>
					{#each Object.keys(scales) as s (s)}
						<option value={s}>{s}</option>
					{/each}
				</select>
			</div>

			<div class="flex items-center gap-2 text-xs text-slate-400">
				<label class="flex cursor-pointer items-center gap-2">
					<input
						type="checkbox"
						bind:checked={allowNonScaleNotes}
						class="rounded border-slate-700 bg-slate-900 text-emerald-500
							focus:ring-2 focus:ring-emerald-500/70 focus:ring-offset-0"
					/>
					<span class="text-slate-300">Allow non-scale notes</span>
				</label>
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
			class="flex gap-2 overflow-hidden rounded-xl border border-slate-800 bg-slate-900/60 p-3 shadow-lg shadow-black/40 {isPainting
				? 'cursor-crosshair'
				: ''}"
		>
			<div class="flex max-h-[600px] flex-col gap-0.5 overflow-y-auto pt-6 pb-6">
				{#each [...Array(NUM_PITCHES).keys()] as pitchIndex (pitchIndex)}
					<div class="flex gap-2">
						<div
							class="flex w-16 flex-shrink-0 items-center justify-end pr-2 text-xs {getNoteLabelClasses(
								pitchIndex
							)}"
						>
							<span>{getNoteName(pitchIndex)}</span>
						</div>
						<div class="flex gap-0.5">
							{#each patterns.find((p) => p.id === displayedPatternId)?.steps || [] as column, stepIndex (stepIndex)}
								{@const stepsPerPat = stepsPerPattern()}
								{@const currentPatternIndex = Math.floor(currentStep / stepsPerPat)}
								{@const currentStepInPattern = currentStep % stepsPerPat}
								{@const isPlayingThisStep =
									isPlaying &&
									currentPatternIndex < patternIdSequence.length &&
									patternIdSequence[currentPatternIndex] === displayedPatternId &&
									currentStepInPattern === stepIndex}
								{@const isDisabled = !allowNonScaleNotes && !isInScale(pitchIndex)}
								<button
									type="button"
									disabled={isDisabled}
									on:mousedown={(e) => {
										if (isDisabled) return;
										e.preventDefault();
										paintTargetState = !column.pitches[pitchIndex];
										setCell(stepIndex, pitchIndex, paintTargetState);
										isPainting = true;
									}}
									on:mouseenter={() => {
										if (isPainting && !isDisabled) {
											setCell(stepIndex, pitchIndex, paintTargetState);
										}
									}}
									class={`h-8 w-8 flex-shrink-0 rounded-sm border text-xs transition
										${getCellClasses(pitchIndex, column.pitches[pitchIndex])}
										${isPlayingThisStep ? 'ring-2 ring-emerald-400/80 ring-offset-2 ring-offset-slate-900' : ''}
										${isDisabled ? 'cursor-not-allowed opacity-40' : 'cursor-pointer'}
									`}
									aria-label="Toggle note"
								></button>
							{/each}
						</div>
					</div>
				{/each}
			</div>
		</section>

		<section class="flex flex-wrap items-center gap-4">
			<div class="flex items-center gap-2 text-xs text-slate-400">
				<span class="text-slate-300">Sequence</span>
				<div class="flex items-center gap-1.5">
					{#each patternIdSequence as patternId, seqIndex (seqIndex)}
						{@const isPlayingThis =
							isPlaying && Math.floor(currentStep / stepsPerPattern()) === seqIndex}
						{@const isSelected = selectedSequenceIndex === seqIndex}
						<div
							role="button"
							tabindex="0"
							on:click={() => {
								selectedSequenceIndex = seqIndex;
							}}
							on:keydown={(e) => {
								if (e.key === 'Enter' || e.key === ' ') {
									e.preventDefault();
									selectedSequenceIndex = seqIndex;
								}
							}}
							class={`flex min-w-[2.5rem] cursor-pointer flex-col items-center rounded-md px-2 py-1 transition
								${getPatternColor(patternId)}
								${isSelected ? 'ring-2 ring-white ring-offset-2 ring-offset-slate-900' : ''}
								${isPlayingThis ? 'shadow-lg shadow-emerald-500/50' : ''}
							`}
						>
							<button
								type="button"
								on:click={(e) => {
									e.stopPropagation();
									cyclePatternAtPosition(seqIndex, 1);
								}}
								class="text-white/80 hover:scale-110 hover:text-white"
								title="Next pattern"
							>
								▲
							</button>
							<span class="text-sm font-semibold text-white">{patternId}</span>
							<button
								type="button"
								on:click={(e) => {
									e.stopPropagation();
									cyclePatternAtPosition(seqIndex, -1);
								}}
								class="text-white/80 hover:scale-110 hover:text-white"
								title="Previous pattern"
							>
								▼
							</button>
						</div>
					{/each}
					{#if patternIdSequence.length < MAX_MEASURES}
						<button
							type="button"
							on:click={() => {
								patternIdSequence.push(PATTERN_IDS[0]);
							}}
							class="rounded-md border border-slate-700 bg-slate-900 px-2 py-1 text-xs hover:bg-slate-800"
							title="Add to sequence"
						>
							+
						</button>
					{/if}
					{#if patternIdSequence.length > 1}
						<button
							type="button"
							on:click={() => {
								patternIdSequence = patternIdSequence.slice(0, -1);
								if (selectedSequenceIndex >= patternIdSequence.length) {
									selectedSequenceIndex = patternIdSequence.length - 1;
									if (selectedSequenceIndex < 0) {
										selectedSequenceIndex = 0;
									}
								}
							}}
							class="rounded-md border border-slate-700 bg-slate-900 px-2 py-1 text-xs hover:bg-slate-800"
							title="Remove last"
						>
							−
						</button>
					{/if}
				</div>
			</div>
		</section>
	</div>
</main>
