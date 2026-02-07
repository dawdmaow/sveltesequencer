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
	const MIN_PATTERN_ID = 0;
	const MAX_PATTERNS = 8;

	const activeNotes = new Map<string, { osc: OscillatorNode; gain: GainNode; releaseAt: number }>();

	type SynthId = number;
	type PatternId = number;

	interface Step {
		pitches: boolean[];
	}

	interface Synth {
		id: SynthId;
		name: string;
		oscType: OscillatorType;
		volume: number;
		attackMs: number;
		releaseMs: number;
		legatoSameNote: boolean;
	}

	interface Pattern {
		id: PatternId;
		beatsPerMeasure: number;
		stepsPerBeat: number;
		layers: Step[][];
	}

	function createStepLayer(stepCount: number): Step[] {
		return Array.from({ length: stepCount }, () => ({
			pitches: Array(NUM_PITCHES).fill(false)
		}));
	}

	function createPattern(patternId: PatternId, beats: number, steps: number): Pattern {
		const clampedBeats = clampBeatsPerMeasure(beats);
		const clampedSteps = clampStepsPerBeat(steps);
		const stepsCount = clampedBeats * clampedSteps;
		return {
			id: patternId,
			beatsPerMeasure: clampedBeats,
			stepsPerBeat: clampedSteps,
			layers: [createStepLayer(stepsCount), createStepLayer(stepsCount)]
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
	let sequence = $state<PatternId[]>([]);
	let selectedSequenceIndex = $state(0);
	let isPlaying = $state(false);
	let currentStep = $state(0); // TODO: this should be local; need to store current sequence index as well (and give it to playStep())
	let bpm = $state(120);
	let stepsPerBeat = $state(4);
	let numMeasures = $state(1);
	let key = $state('C');
	let scale = $state<Scale>('Natural Minor');
	let allowNonScaleNotes = $state(true);
	let synths = $state<Synth[]>([]);
	let selectedSynthIndex = $state(0);
	let isPainting = $state(false);
	let paintTargetState = $state(true);
	let selectedNotes = $state<Set<string>>(new Set()); // js is dumb, sets with non-primitive types assume ref semantics (so we use string as a substitute)
	let selectionRectAnchor = $state<{ step: number; pitch: number } | null>(null);
	let selectionRectExtent = $state<{ step: number; pitch: number } | null>(null);
	let isSelectingRect = $state(false);
	let isDragging = $state(false);
	let dragStartCell = $state<{ step: number; pitch: number } | null>(null);
	let dragStartNotes = $state<Array<{ step: number; pitch: number }>>([]);
	let dragLastPositions = $state<Array<{ step: number; pitch: number }>>([]);
	let dragOverwrittenNotes = $state<Array<{ step: number; pitch: number }>>([]);
	let isCloning = $state(false);

	let selectedPatternId = $derived(sequence[selectedSequenceIndex] ?? MIN_PATTERN_ID);

	let displayedPatternId = $derived(
		isPlaying ? (getPatternAtStep(currentStep)?.patternId ?? selectedPatternId) : selectedPatternId
	);

	let displayedPattern = $derived(patterns.find((p) => p.id === displayedPatternId) ?? null);

	function getCurrentLayer(): Step[] | null {
		const pattern = patterns.find((p) => p.id === displayedPatternId);
		if (!pattern || selectedSynthIndex >= pattern.layers.length) return null;
		return pattern.layers[selectedSynthIndex];
	}

	function reset() {
		patterns = Array.from({ length: MAX_PATTERNS }, (_, i) =>
			createPattern(MIN_PATTERN_ID + i, 4, 4)
		);
		sequence = [0];
		selectedSequenceIndex = 0;
		selectedNotes = new Set();
		key = 'C';
		scale = 'Natural Minor';
		allowNonScaleNotes = true;
		synths = [
			{
				id: 0,
				name: 'Synth 1',
				oscType: 'square',
				volume: 0.3,
				attackMs: 10,
				releaseMs: 100,
				legatoSameNote: false
			},
			{
				id: 1,
				name: 'Synth 2',
				oscType: 'sine',
				volume: 0.2,
				attackMs: 5,
				releaseMs: 80,
				legatoSameNote: false
			}
		];
		selectedSynthIndex = 0;
		bpm = 120;
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

	function stepsPerPattern(patternId: PatternId): number {
		const pattern = patterns.find((p) => p.id === patternId);
		if (!pattern) return 4 * 4;
		return clampBeatsPerMeasure(pattern.beatsPerMeasure) * clampStepsPerBeat(pattern.stepsPerBeat);
	}

	function getStepForSequenceIndex(seqIndex: number): number {
		let sum = 0;
		for (let i = 0; i < seqIndex && i < sequence.length; i++) {
			sum += stepsPerPattern(sequence[i]);
		}
		return sum;
	}

	function getPatternAtStep(
		globalStep: number
	): { patternIndex: number; stepInPattern: number; patternId: PatternId } | null {
		let acc = 0;
		for (let i = 0; i < sequence.length; i++) {
			const patSteps = stepsPerPattern(sequence[i]);
			if (globalStep < acc + patSteps) {
				return {
					patternIndex: i,
					stepInPattern: globalStep - acc,
					patternId: sequence[i]
				};
			}
			acc += patSteps;
		}
		return null;
	}

	function getPatternColor(patternId: PatternId): string {
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
		const index = patternId;
		return index >= 0 && index < colors.length ? colors[index] : 'bg-slate-600';
	}

	function cyclePatternAtPosition(seqIndex: number, direction: 1 | -1) {
		if (seqIndex >= sequence.length) return;
		const currentId = sequence[seqIndex];
		const nextId =
			((currentId - MIN_PATTERN_ID + direction + MAX_PATTERNS) % MAX_PATTERNS) + MIN_PATTERN_ID;
		sequence[seqIndex] = nextId;
		selectedSequenceIndex = seqIndex;
	}

	function movePatternInSequence(seqIndex: number, direction: -1 | 1) {
		if (direction === -1 && seqIndex === 0) return;
		if (direction === 1 && seqIndex >= sequence.length - 1) return;
		const other = seqIndex + direction;
		const next = [...sequence];
		[next[seqIndex], next[other]] = [next[other], next[seqIndex]];
		sequence = next;
		selectedSequenceIndex = other;
		setTimeout(() => document.getElementById(`seq-cell-${other}`)?.focus(), 0);
	}

	function totalSteps(): number {
		return sequence.reduce((sum, patternId) => sum + stepsPerPattern(patternId), 0);
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

	function playNote(midiNote: number, time: number, duration: number, synth: Synth) {
		const ctx = ensureAudioContext();
		const key = `${synth.id}-${midiNote}`;
		if (synth.legatoSameNote) {
			const existing = activeNotes.get(key);
			if (existing && ctx.currentTime < existing.releaseAt - 0.02) {
				existing.gain.gain.cancelScheduledValues(time);
				existing.gain.gain.setValueAtTime(existing.gain.gain.value, time);
				const releaseSec = synth.releaseMs / 1000;
				existing.gain.gain.linearRampToValueAtTime(synth.volume, time + 0.005);
				existing.gain.gain.linearRampToValueAtTime(synth.volume, time + duration - releaseSec);
				existing.gain.gain.linearRampToValueAtTime(0, time + duration);
				existing.releaseAt = time + duration;
				return;
			}
			if (existing) {
				existing.osc.stop();
				existing.gain.disconnect();
				activeNotes.delete(key);
				// TODO: if we never play the same synth-note combination, the key in activeNotes will stay forever (and oscillator will keep running)
			}
		}
		const osc = ctx.createOscillator();
		const gain = ctx.createGain();

		osc.type = synth.oscType;
		osc.frequency.value = midiToFreq(midiNote);

		gain.gain.setValueAtTime(0, time);
		const attackSec = synth.attackMs / 1000;
		const releaseSec = synth.releaseMs / 1000;
		gain.gain.linearRampToValueAtTime(synth.volume, time + attackSec);
		const releaseStart = Math.max(time + attackSec, time + duration - releaseSec);
		gain.gain.linearRampToValueAtTime(synth.volume, releaseStart);
		gain.gain.linearRampToValueAtTime(0, time + duration);

		osc.connect(gain);
		gain.connect(ctx.destination);

		osc.start(time);
		if (synth.legatoSameNote) {
			activeNotes.set(key, { osc, gain, releaseAt: time + duration });
		} else {
			osc.stop(time + duration + 0.05);
		}
	}

	function setCell(step: number, pitchIndex: number, state: boolean) {
		if (!allowNonScaleNotes && !isInScale(pitchIndex)) return;
		const layer = getCurrentLayer();
		if (!layer || step >= layer.length) return;
		layer[step].pitches[pitchIndex] = state;
	}

	function isNoteSelected(step: number, pitch: number): boolean {
		return selectedNotes.has(`${step}-${pitch}`);
	}

	function getSelectionRect(): {
		minStep: number;
		maxStep: number;
		minPitch: number;
		maxPitch: number;
	} | null {
		if (!selectionRectAnchor || !selectionRectExtent) return null;
		return {
			minStep: Math.min(selectionRectAnchor.step, selectionRectExtent.step),
			maxStep: Math.max(selectionRectAnchor.step, selectionRectExtent.step),
			minPitch: Math.min(selectionRectAnchor.pitch, selectionRectExtent.pitch),
			maxPitch: Math.max(selectionRectAnchor.pitch, selectionRectExtent.pitch)
		};
	}

	function isInSelectionRect(step: number, pitch: number): boolean {
		const rect = getSelectionRect();
		if (!rect) return false;
		return (
			step >= rect.minStep &&
			step <= rect.maxStep &&
			pitch >= rect.minPitch &&
			pitch <= rect.maxPitch
		);
	}

	function addNotesInRectToSelection(rect: {
		minStep: number;
		maxStep: number;
		minPitch: number;
		maxPitch: number;
	}) {
		const layer = getCurrentLayer();
		if (!layer) return;
		const next = new Set(selectedNotes);
		for (let s = rect.minStep; s <= rect.maxStep; s++) {
			for (let p = rect.minPitch; p <= rect.maxPitch; p++) {
				if (layer[s]?.pitches[p]) {
					next.add(`${s}-${p}`);
				}
			}
		}
		selectedNotes = next;
	}

	function getSelectedNotesArray(): Array<{ step: number; pitch: number }> {
		return Array.from(selectedNotes).map((k) => {
			const [s, p] = k.split('-');
			return { step: +s, pitch: +p };
		});
	}

	function clearNotesAt(positions: Array<{ step: number; pitch: number }>) {
		const layer = getCurrentLayer();
		if (!layer) return;
		for (const { step, pitch } of positions) {
			if (layer[step]) {
				layer[step].pitches[pitch] = false;
			}
		}
	}

	function setNotesAt(positions: Array<{ step: number; pitch: number }>) {
		const layer = getCurrentLayer();
		if (!layer) return;
		for (const { step, pitch } of positions) {
			if (step >= 0 && step < layer.length && pitch >= 0 && pitch < NUM_PITCHES) {
				layer[step].pitches[pitch] = true;
			}
		}
	}

	function moveSelectedNotes(deltaStep: number, deltaPitch: number) {
		const notes = getSelectedNotesArray();
		const layer = getCurrentLayer();
		if (notes.length === 0 || !layer) {
			return;
		}
		const newPositions = notes.map((n) => ({
			step: n.step + deltaStep,
			pitch: n.pitch + deltaPitch
		}));
		const inBounds = newPositions.every(
			(p) => p.step >= 0 && p.step < layer.length && p.pitch >= 0 && p.pitch < NUM_PITCHES
		);
		if (!inBounds) {
			return;
		}
		clearNotesAt(notes);
		setNotesAt(newPositions);
		selectedNotes = new Set(newPositions.map((p) => `${p.step}-${p.pitch}`));
	}

	function handleCellMousedown(
		stepIndex: number,
		pitchIndex: number,
		e: MouseEvent,
		opts: { hasNote: boolean; noteSelected: boolean; isDisabled: boolean }
	) {
		if (opts.isDisabled) return;
		e.preventDefault();
		if (e.metaKey || e.ctrlKey) {
			selectionRectAnchor = { step: stepIndex, pitch: pitchIndex };
			selectionRectExtent = { step: stepIndex, pitch: pitchIndex };
			isSelectingRect = true;
		} else if (e.shiftKey && opts.hasNote) {
			const key = `${stepIndex}-${pitchIndex}`;
			const next = new Set(selectedNotes);
			if (next.has(key)) next.delete(key);
			else next.add(key);
			selectedNotes = next;
		} else if (opts.noteSelected && selectedNotes.size > 0) {
			const notes = getSelectedNotesArray();
			selectedNotes = new Set();
			dragStartNotes = notes;
			dragStartCell = { step: stepIndex, pitch: pitchIndex };
			dragLastPositions = [];
			dragOverwrittenNotes = [];
			isCloning = e.altKey;
			isDragging = true;
		} else {
			selectedNotes = new Set();
			paintTargetState = !opts.hasNote;
			setCell(stepIndex, pitchIndex, paintTargetState);
			isPainting = true;
		}
	}

	function handleCellMouseenter(stepIndex: number, pitchIndex: number, isDisabled: boolean) {
		if (isSelectingRect) {
			selectionRectExtent = { step: stepIndex, pitch: pitchIndex };
		} else if (isDragging && dragStartCell) {
			// NOTE: we are dragging the notes

			const deltaStep = stepIndex - dragStartCell.step;
			const deltaPitch = pitchIndex - dragStartCell.pitch;
			const layer = getCurrentLayer();
			if (!layer) {
				return;
			}
			const stepsLen = layer.length;

			// NOTE: we remove notes that are are currently being grabbed
			if (isCloning) {
				clearNotesAt(dragLastPositions);
			} else {
				if (dragLastPositions.length > 0) {
					clearNotesAt(dragLastPositions);
				} else {
					clearNotesAt(dragStartNotes); // NOTE: we do this because lastPositions being empty implies we have just started the drag
				}
			}

			// NOTE: we set notes that are being overwritten (by the grabbed notes)
			setNotesAt(dragOverwrittenNotes);

			// NOTE: we calculate the new positions of the grabbed notes
			const newLastPositions = dragStartNotes
				.map((n) => ({
					step: n.step + deltaStep,
					pitch: n.pitch + deltaPitch
				}))
				.filter((n) => n.step >= 0 && n.step < stepsLen && n.pitch >= 0 && n.pitch < NUM_PITCHES);

			// NOTE: we calculate the notes that are being overwritten (by the grabbed notes)
			const newOverwrittenNotes = newLastPositions.filter((p) => layer[p.step]?.pitches[p.pitch]);

			setNotesAt(newLastPositions);
			dragLastPositions = newLastPositions;
			dragOverwrittenNotes = newOverwrittenNotes;
		} else if (isPainting && !isDisabled) {
			setCell(stepIndex, pitchIndex, paintTargetState);
		}
	}

	function stepDurationSeconds() {
		const numericBpm = clampBpm(Number(bpm));
		const steps = clampStepsPerBeat(stepsPerBeat);
		return 60 / (numericBpm * steps);
	}

	function playStep(step: number) {
		const ctx = ensureAudioContext();
		const time = ctx.currentTime;
		const stepDur = stepDurationSeconds();
		const at = getPatternAtStep(step);
		if (!at) return;

		const { patternId, stepInPattern } = at;
		const pattern = patterns.find((p) => p.id === patternId);
		if (!pattern) {
			return;
		}

		for (let synthIndex = 0; synthIndex < synths.length; synthIndex++) {
			const layer = pattern.layers[synthIndex];
			const synth = synths[synthIndex];
			if (!layer || stepInPattern >= layer.length || !synth) {
				continue;
			}
			const duration = synth.legatoSameNote ? stepDur * 2 : stepDur * 0.9;
			const pitches = layer[stepInPattern].pitches;
			for (const [pitchIndex, active] of pitches.entries()) {
				if (active) {
					const midi = BASE_MIDI_NOTE + (NUM_PITCHES - 1 - pitchIndex);
					playNote(midi, time, duration, synth);
				}
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

		currentStep = getStepForSequenceIndex(selectedSequenceIndex);
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
		for (const { osc, gain } of activeNotes.values()) {
			osc.stop();
			gain.disconnect();
		}
		activeNotes.clear();
	}

	function clearPattern() {
		const pattern = patterns.find((p) => p.id === displayedPatternId);
		if (!pattern) return;
		const patternIndex = patterns.findIndex((p) => p.id === displayedPatternId);
		patterns[patternIndex] = createPattern(
			displayedPatternId,
			pattern.beatsPerMeasure,
			pattern.stepsPerBeat
		);
	}

	$effect(() => {
		if (typeof document === 'undefined') return;
		document.body.style.cursor = isDragging ? 'grabbing' : '';
		return () => {
			document.body.style.cursor = '';
		};
	});

	$effect(() => {
		for (const pattern of patterns) {
			const stepsPerPat = stepsPerPattern(pattern.id);
			for (const layer of pattern.layers) {
				if (layer.length !== stepsPerPat) {
					if (stepsPerPat > layer.length) {
						for (let i = layer.length; i < stepsPerPat; i++) {
							layer.push({ pitches: Array(NUM_PITCHES).fill(false) });
						}
					} else {
						layer.length = stepsPerPat;
					}
				}
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
		if (e.target instanceof HTMLInputElement || e.target instanceof HTMLTextAreaElement) {
			return;
		}
		if (e.key === 'Delete' || e.key === 'Backspace') {
			if (selectedNotes.size > 0) {
				e.preventDefault();
				clearNotesAt(getSelectedNotesArray());
				selectedNotes = new Set();
			}
		} else if (e.key === 'Escape') {
			if (selectedNotes.size > 0 || isSelectingRect) {
				// TODO: this doesnt close selecting rect, probably due to some preventDefault
				e.preventDefault();
				selectedNotes = new Set();
				selectionRectAnchor = null;
				selectionRectExtent = null;
				isSelectingRect = false;
			}
		} else if (
			selectedNotes.size > 0 &&
			(e.key === 'ArrowUp' ||
				e.key === 'ArrowDown' ||
				e.key === 'ArrowLeft' ||
				e.key === 'ArrowRight')
		) {
			e.preventDefault();
			let deltaStep = 0;
			let deltaPitch = 0;
			if (e.key === 'ArrowUp') deltaPitch = -1;
			else if (e.key === 'ArrowDown') deltaPitch = 1;
			else if (e.key === 'ArrowLeft') deltaStep = -1;
			else if (e.key === 'ArrowRight') deltaStep = 1;
			moveSelectedNotes(deltaStep, deltaPitch);
		} else if (e.key === ' ') {
			e.preventDefault();
			if (isPlaying) {
				stop();
			} else {
				start();
			}
		}
	}}
	on:mouseup={() => {
		if (isSelectingRect) {
			const rect = getSelectionRect();
			if (rect) {
				addNotesInRectToSelection(rect);
			}
			selectionRectAnchor = null;
			selectionRectExtent = null;
			isSelectingRect = false;
		}
		isPainting = false;
		isDragging = false;
		isCloning = false;
		dragStartCell = null;
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
				<label for="gridSteps" class="text-slate-300">Grid steps/beat</label>
				<input
					id="gridSteps"
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

			<div class="flex items-center gap-1">
				{#each synths as synth, idx (synth.id)}
					<button
						type="button"
						on:click={() => {
							selectedSynthIndex = idx;
							selectedNotes = new Set();
						}}
						class="rounded-md px-3 py-1.5 text-sm transition
							{selectedSynthIndex === idx
							? 'bg-emerald-600 text-white'
							: 'bg-slate-800 text-slate-400 hover:bg-slate-700 hover:text-slate-300'}"
					>
						{synth.name}
					</button>
				{/each}
			</div>

			<div class="flex items-center gap-2 text-xs text-slate-400">
				<label for="osc" class="text-slate-300">Osc</label>
				<select
					id="osc"
					bind:value={synths[selectedSynthIndex].oscType}
					class="rounded-md border border-slate-700 bg-slate-900 px-2 py-1 text-sm
						focus-visible:ring-2 focus-visible:ring-emerald-500/70 focus-visible:outline-none"
				>
					<option value="sine">sine</option>
					<option value="square">square</option>
					<option value="sawtooth">sawtooth</option>
					<option value="triangle">triangle</option>
				</select>
			</div>

			<div class="flex items-center gap-2 text-sm">
				<label for="vol" class="text-slate-300">Vol</label>
				<input
					id="vol"
					type="range"
					min="0"
					max="1"
					step="0.01"
					bind:value={synths[selectedSynthIndex].volume}
					class="w-20 accent-emerald-500"
				/>
				<span class="w-8 text-right text-slate-400"
					>{Math.round(synths[selectedSynthIndex].volume * 100)}%</span
				>
			</div>

			<div class="flex items-center gap-2 text-sm">
				<label for="attack" class="text-slate-300">Attack (ms)</label>
				<input
					id="attack"
					type="range"
					min="1"
					max="100"
					step="1"
					bind:value={synths[selectedSynthIndex].attackMs}
					class="w-20 accent-emerald-500"
				/>
				<span class="min-w-12 shrink-0 text-right whitespace-nowrap text-slate-400"
					>{synths[selectedSynthIndex].attackMs} ms</span
				>
			</div>

			<div class="flex items-center gap-2 text-sm">
				<label for="release" class="text-slate-300">Release (ms)</label>
				<input
					id="release"
					type="range"
					min="1"
					max="500"
					step="1"
					bind:value={synths[selectedSynthIndex].releaseMs}
					class="w-20 accent-emerald-500"
				/>
				<span class="min-w-12 shrink-0 text-right whitespace-nowrap text-slate-400"
					>{synths[selectedSynthIndex].releaseMs} ms</span
				>
			</div>

			<div class="flex items-center gap-2 text-sm">
				<label class="flex cursor-pointer items-center gap-2 text-slate-300">
					<input
						type="checkbox"
						bind:checked={synths[selectedSynthIndex].legatoSameNote}
						class="rounded"
					/>
					Legato
				</label>
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

			{#if displayedPattern}
				<div class="flex items-center gap-2 text-sm">
					<label for="beats" class="text-slate-300">Beats / bar</label>
					<input
						id="beats"
						type="number"
						min={MIN_BEATS_PER_MEASURE}
						max={MAX_BEATS_PER_MEASURE}
						bind:value={displayedPattern.beatsPerMeasure}
						on:blur={() => {
							displayedPattern.beatsPerMeasure = clampBeatsPerMeasure(
								displayedPattern.beatsPerMeasure
							);
						}}
						on:keydown={(event) => {
							if (event.key === 'Enter') {
								displayedPattern.beatsPerMeasure = clampBeatsPerMeasure(
									displayedPattern.beatsPerMeasure
								);
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
						bind:value={displayedPattern.stepsPerBeat}
						on:blur={() => {
							displayedPattern.stepsPerBeat = clampStepsPerBeat(displayedPattern.stepsPerBeat);
						}}
						on:keydown={(event) => {
							if (event.key === 'Enter') {
								displayedPattern.stepsPerBeat = clampStepsPerBeat(displayedPattern.stepsPerBeat);
							}
						}}
						class="w-24 rounded-md border border-slate-700 bg-slate-900 px-2 py-1 text-right text-sm
							focus-visible:ring-2 focus-visible:ring-emerald-500/70 focus-visible:outline-none"
					/>
				</div>
			{/if}
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
							{#each getCurrentLayer() || [] as column, stepIndex (stepIndex)}
								{@const at = getPatternAtStep(currentStep)}
								{@const isPlayingThisStep =
									isPlaying &&
									at !== null &&
									at.patternId === displayedPatternId &&
									at.stepInPattern === stepIndex}
								{@const isDisabled = !allowNonScaleNotes && !isInScale(pitchIndex)}
								{@const hasNote = column.pitches[pitchIndex]}
								{@const noteSelected = isNoteSelected(stepIndex, pitchIndex)}
								{@const inSelectionRect = isInSelectionRect(stepIndex, pitchIndex)}
								<button
									type="button"
									disabled={isDisabled}
									on:mousedown={(e) =>
										handleCellMousedown(stepIndex, pitchIndex, e, {
											hasNote,
											noteSelected,
											isDisabled
										})}
									on:mouseenter={() => handleCellMouseenter(stepIndex, pitchIndex, isDisabled)}
									class={`h-8 w-8 flex-shrink-0 rounded-sm border text-xs transition
										${getCellClasses(pitchIndex, column.pitches[pitchIndex])}
										${isPlayingThisStep ? 'ring-2 ring-emerald-400/80 ring-offset-2 ring-offset-slate-900' : ''}
										${isSelectingRect && inSelectionRect ? 'brightness-150' : ''}
										${noteSelected ? 'brightness-125' : ''}
										${isDisabled ? 'cursor-not-allowed opacity-40' : noteSelected ? 'cursor-grab' : 'cursor-pointer'}
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
					{#each sequence as patternId, seqIndex (seqIndex)}
						{@const isPlayingThis =
							isPlaying && getPatternAtStep(currentStep)?.patternIndex === seqIndex}
						{@const isSelected = selectedSequenceIndex === seqIndex}
						<div
							id="seq-cell-{seqIndex}"
							role="button"
							tabindex="0"
							on:click={() => {
								selectedSequenceIndex = seqIndex;
							}}
							on:keydown={(e) => {
								if (e.key === 'Enter' || e.key === ' ') {
									e.preventDefault();
									selectedSequenceIndex = seqIndex;
								} else if (
									e.key === 'ArrowUp' ||
									e.key === 'ArrowDown' ||
									e.key === 'ArrowLeft' ||
									e.key === 'ArrowRight'
								) {
									e.preventDefault();
									e.stopPropagation();
									if (e.key === 'ArrowUp') cyclePatternAtPosition(seqIndex, 1);
									else if (e.key === 'ArrowDown') cyclePatternAtPosition(seqIndex, -1);
									else if (e.key === 'ArrowLeft') movePatternInSequence(seqIndex, -1);
									else if (e.key === 'ArrowRight') movePatternInSequence(seqIndex, 1);
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
							<span class="text-sm font-semibold text-white">{patternId + 1}</span>
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
					{#if sequence.length < MAX_MEASURES}
						<button
							type="button"
							on:click={() => {
								sequence.push(selectedPatternId);
							}}
							class="rounded-md border border-slate-700 bg-slate-900 px-2 py-1 text-xs hover:bg-slate-800"
							title="Add to sequence"
						>
							+
						</button>
					{/if}
					{#if sequence.length > 1}
						<button
							type="button"
							on:click={() => {
								sequence = sequence.slice(0, -1);
								if (selectedSequenceIndex >= sequence.length) {
									selectedSequenceIndex = sequence.length - 1;
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
