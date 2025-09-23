<script setup lang="ts">
import { ref, onMounted } from "vue";
import { Chord } from "tonal";
import Soundfont from "soundfont-player";
import { Instrument } from "piano-chart";

const pianoContainer = ref<HTMLDivElement | null>(null);

onMounted(() => {
    if (!pianoContainer.value) return;

    const piano = new Instrument(pianoContainer.value, {
        startOctave: 3,
        endOctave: 5,
        highlightedNotes: ["D", "E", "F#", "G", "A", "B", "C#"],
        specialHighlightedNotes: [{ note: "D" }],
    });

piano.create();

piano.keyDown("D4");
piano.keyDown("F#4");
piano.keyDown("A4");



});



const rows = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"];
const cols = [
    { label: "Majeur", suffix: "" },
    { label: "Mineur", suffix: "m" },
    { label: "Maj7", suffix: "maj7" },
    { label: "Min7", suffix: "m7" },
    { label: "Deuxième", suffix: "sus2" },
    { label: "Quatrième", suffix: "sus4" },
    { label: "Augmenté", suffix: "aug" },
    { label: "Diminué", suffix: "dim" }
];

// 🎵 Modes disponibles
const playModes = ["Accord", "Arpège", "Random", "Descendant", "Montant"];
const selectedMode = ref("Arpège");

let audioCtx: AudioContext | null = null;
let pianoInstrument: any = null;

// Charger le piano Soundfont
async function loadPiano() {
    if (!audioCtx) {
        audioCtx = new AudioContext();
    }
    pianoInstrument = await Soundfont.instrument(audioCtx, "acoustic_grand_piano");
}
loadPiano();

// Jouer un accord
async function playChord(root: string, suffix: string) {
    if (!pianoInstrument || !audioCtx) return;
    if (audioCtx.state === "suspended") {
        await audioCtx.resume();
    }

    const chord = Chord.get(root + suffix);
    if (!chord.empty) {
        let notes = chord.notes.map((n) => n + "4");

        switch (selectedMode.value) {
            case "Accord":
                // toutes en même temps
                notes.forEach((note) => pianoInstrument.play(note, audioCtx.currentTime, { duration: 0.6 }));
                break;

            case "Arpège":
                // de bas en haut
                notes.forEach((note, i) =>
                    pianoInstrument.play(note, audioCtx.currentTime + i * 0.2)
                );
                break;

            case "Random":
                // ordre aléatoire
                notes = [...notes].sort(() => Math.random() - 0.5);
                notes.forEach((note, i) =>
                    pianoInstrument.play(note, audioCtx.currentTime + i * 0.2)
                );
                break;

            case "Descendant":
                // de haut en bas
                notes
                    .slice()
                    .reverse()
                    .forEach((note, i) =>
                        pianoInstrument.play(note, audioCtx.currentTime + i * 0.2)
                    );
                break;
            case "Montant":
                // de haut en bas
                notes
                    .slice()
                    .forEach((note, i) =>
                        pianoInstrument.play(note, audioCtx.currentTime + i * 0.2)
                    );
                break;
        }
    }
}
</script>

<template>
    <div class="p-4">
        <h2 class="title">Accords principaux au piano</h2>

        <!-- Sélecteur de mode -->
        <div class="mb-4">
            <label class="mr-2 font-semibold">Mode :</label>
            <select v-model="selectedMode" class="border rounded p-1">
                <option v-for="mode in playModes" :key="mode">{{ mode }}</option>
            </select>
            <div ref="pianoContainer"></div>
        </div>

        <table class="chords-table">
            <thead>
                <tr>
                    <th></th>
                    <th v-for="col in cols" :key="col.label">{{ col.label }}</th>
                </tr>
            </thead>
            <tbody>
                <tr v-for="row in rows" :key="row">
                    <td class="row-label">{{ row }}</td>
                    <td v-for="col in cols" :key="row + col.label" class="chord-cell"
                        @click="playChord(row, col.suffix)">
                        {{ row + col.suffix }}
                    </td>
                </tr>
            </tbody>
        </table>
    </div>
</template>

<style scoped>
.title {
    font-size: 1.25rem;
    font-weight: bold;
    margin-bottom: 1rem;
}

.chords-table {
    border-collapse: collapse;
    width: 100%;
    text-align: center;
}

.chords-table th,
.chords-table td {
    border: 1px solid #999;
    padding: 8px;
    cursor: pointer;
}

.row-label {
    font-weight: bold;
    background: #f3f4f6;
}

.chord-cell:hover {
    background: #facc15;
    /* jaune au survol */
}
</style>
