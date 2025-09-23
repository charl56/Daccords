<template>
    <div class="daccords-div bg-gradient-to-br from-indigo-50 to-blue-100">
        <button v-if="!audioInitialized" @click="initializeAudio" class="btn btn-activation-audio">
            🔊 Activer l'audio
        </button>
        <div v-else class="max-w-7xl mx-auto">
            <!-- Piano virtuel -->
            <div class="piano-div">
                <h2 class="text-xl font-semibold text-gray-800 mb-4">
                    Accord actuel: {{ getChordDisplayName(selectedRoot, selectedChord) }}
                </h2>
                <PianoComponent :highlighted-notes="currentChordNotes" @note-click="playNote" />
                <div class="mt-4 text-center">
                    <div class="text-sm text-gray-600 mb-2">
                        Notes: {{currentChordNotes.map(note => [note]).join(' - ')}}
                    </div>
                    <button @click="playChord(selectedRoot, selectedChord)" :disabled="isPlaying || !audioInitialized"
                        class="px-6 py-2 rounded-lg font-semibold text-white transition-all duration-200 flex items-center gap-2 mx-auto"
                        :class="[
                            isPlaying || !audioInitialized
                                ? 'bg-gray-400 cursor-not-allowed'
                                : 'bg-indigo-600 hover:bg-indigo-700 active:scale-95'
                        ]">
                        {{ isPlaying ? '⏸️ Lecture...' : '▶️ Jouer l\'accord' }}
                    </button>
                </div>
            </div>

            <!-- Contrôles -->
            <div class="bg-white rounded-xl shadow-lg p-6 mb-6">
                <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
                    <!-- Mode de lecture -->
                    <div>
                        <label class="block text-sm font-semibold text-gray-700 mb-2">Mode de lecture</label>
                        <select v-model="playMode.type"
                            class="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-transparent">
                            <option value="simultané">Simultané</option>
                            <option value="arpège montant">Arpège montant</option>
                            <option value="arpège descendant">Arpège descendant</option>
                            <option value="arpège alterné">Arpège alterné</option>
                        </select>
                    </div>

                    <!-- Vitesse -->
                    <div>
                        <label class="block text-sm font-semibold text-gray-700 mb-2">
                            Vitesse arpège: {{ playMode.speed }}ms
                        </label>
                        <input type="range" min="50" max="500" step="50" v-model.number="playMode.speed"
                            class="w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer slider" />
                    </div>

                    <!-- Filtres -->
                    <div class="space-y-3">
                        <div>
                            <label class="block text-sm font-semibold text-gray-700 mb-1">Type</label>
                            <select v-model="filters.type" class="w-full p-2 border border-gray-300 rounded-lg text-sm">
                                <option value="all">Tous les types</option>
                                <option value="major">Majeur</option>
                                <option value="minor">Mineur</option>
                                <option value="augmented">Augmenté</option>
                                <option value="diminished">Diminué</option>
                                <option value="suspended">Suspendu</option>
                                <option value="extended">Étendu</option>
                            </select>
                        </div>
                        <div>
                            <label class="block text-sm font-semibold text-gray-700 mb-1">Humeur</label>
                            <select v-model="filters.mood" class="w-full p-2 border border-gray-300 rounded-lg text-sm">
                                <option value="all">Toutes les humeurs</option>
                                <option value="joyeux">Joyeux</option>
                                <option value="triste">Triste</option>
                                <option value="tendu">Tendu</option>
                                <option value="mystérieux">Mystérieux</option>
                                <option value="neutre">Neutre</option>
                            </select>
                        </div>
                    </div>
                </div>
            </div>


            <!-- Tableau des accords -->
            <div class="bg-white rounded-xl shadow-lg overflow-hidden">
                <div class="p-4 bg-gray-50 border-b">
                    <h2 class="text-xl font-semibold text-gray-800">
                        Tableau des accords ({{ filteredChords.length }} accords)
                    </h2>
                </div>

                <div class="overflow-x-auto">
                    <table class="w-full">
                        <thead class="bg-gray-50">
                            <tr>
                                <th
                                    class="sticky left-0 bg-gray-50 px-4 py-3 text-left font-semibold text-gray-700 border-r">
                                    Note
                                </th>
                                <th v-for="[chordType] in filteredChords" :key="chordType"
                                    class="px-3 py-3 text-center font-semibold text-gray-700 min-w-[80px] border-r">
                                    {{ chordType || 'Maj' }}
                                </th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="(root, rowIndex) in CHROMATIC_NOTES" :key="root"
                                :class="rowIndex % 2 === 0 ? 'bg-gray-50' : 'bg-white'">
                                <td class="sticky left-0 bg-inherit px-4 py-2 font-semibold text-gray-700 border-r">
                                    {{ NOTE_NAMES[root] }}
                                </td>
                                <td v-for="[chordType, chordData] in filteredChords" :key="`${root}-${chordType}`"
                                    class="px-1 py-1 border-r">
                                    <button @click="onChordClick(root, chordType)" :disabled="!audioInitialized"
                                        class="w-full h-12 rounded-md text-xs font-semibold transition-all duration-200 border-2"
                                        :class="[
                                            selectedRoot === root && selectedChord === chordType
                                                ? 'ring-2 ring-indigo-500 ring-offset-2 transform scale-105'
                                                : '',
                                            !audioInitialized
                                                ? 'opacity-50 cursor-not-allowed'
                                                : 'hover:shadow-md active:scale-95 cursor-pointer',
                                            getMoodColor(chordData.mood)
                                        ]" :title="`${getChordDisplayName(root, chordType)} - ${chordData.mood}`">
                                        <div class="flex flex-col items-center justify-center h-full">
                                            <span class="text-gray-800">{{ getChordDisplayName(root, chordType)
                                                }}</span>
                                            <!-- <span class="text-xs opacity-75 capitalize">{{ chordData.mood }}</span> -->
                                        </div>
                                    </button>
                                </td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </div>

            <!-- Légende des humeurs -->
            <div class="mt-6 bg-white rounded-xl shadow-lg p-4">
                <h3 class="text-lg font-semibold text-gray-800 mb-3">Légende des humeurs</h3>
                <div class="flex flex-wrap gap-4">
                    <div class="flex items-center gap-2">
                        <div class="w-4 h-4 rounded joyeux"></div>
                        <span class="text-sm text-gray-700">Joyeux</span>
                    </div>
                    <div class="flex items-center gap-2">
                        <div class="w-4 h-4 rounded triste"></div>
                        <span class="text-sm text-gray-700">Triste</span>
                    </div>
                    <div class="flex items-center gap-2">
                        <div class="w-4 h-4 rounded tendu"></div>
                        <span class="text-sm text-gray-700">Tendu</span>
                    </div>
                    <div class="flex items-center gap-2">
                        <div class="w-4 h-4 rounded mysterieux"></div>
                        <span class="text-sm text-gray-700">Mystérieux</span>
                    </div>
                    <div class="flex items-center gap-2">
                        <div class="w-4 h-4 rounded neutre"></div>
                        <span class="text-sm text-gray-700">Neutre</span>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, computed, onUnmounted, type Ref } from 'vue'
import * as Tone from 'tone'
import PianoComponent from './PianoComponent.vue'

// Types et interfaces
interface ChordData {
    notes: string[]
    intervals: string[]
    type: 'major' | 'minor' | 'augmented' | 'diminished' | 'suspended' | 'extended' | 'other'
    mood: 'joyeux' | 'triste' | 'tendu' | 'mystérieux' | 'neutre'
}

interface PlayMode {
    type: 'simultané' | 'arpège montant' | 'arpège descendant' | 'arpège alterné'
    speed: number
}

interface Filters {
    type: 'all' | 'major' | 'minor' | 'augmented' | 'diminished' | 'suspended' | 'extended' | 'other'
    mood: 'all' | 'joyeux' | 'triste' | 'tendu' | 'mystérieux' | 'neutre'
}

// Données des accords avec classification par humeur
const CHORD_DEFINITIONS: Record<string, ChordData> = {
    '': { notes: ['1', '3', '5'], intervals: ['1', '3', '5'], type: 'major', mood: 'joyeux' },
    'm': { notes: ['1', 'b3', '5'], intervals: ['1', 'b3', '5'], type: 'minor', mood: 'triste' },
    'aug': { notes: ['1', '3', '#5'], intervals: ['1', '3', '#5'], type: 'augmented', mood: 'tendu' },
    'dim': { notes: ['1', 'b3', 'b5'], intervals: ['1', 'b3', 'b5'], type: 'diminished', mood: 'mystérieux' },
    'sus2': { notes: ['1', '2', '5'], intervals: ['1', '2', '5'], type: 'suspended', mood: 'neutre' },
    'sus4': { notes: ['1', '4', '5'], intervals: ['1', '4', '5'], type: 'suspended', mood: 'tendu' },
    '7': { notes: ['1', '3', '5', 'b7'], intervals: ['1', '3', '5', 'b7'], type: 'extended', mood: 'joyeux' },
    'maj7': { notes: ['1', '3', '5', '7'], intervals: ['1', '3', '5', '7'], type: 'extended', mood: 'joyeux' },
    'm7': { notes: ['1', 'b3', '5', 'b7'], intervals: ['1', 'b3', '5', 'b7'], type: 'extended', mood: 'triste' },
    'mM7': { notes: ['1', 'b3', '5', '7'], intervals: ['1', 'b3', '5', '7'], type: 'extended', mood: 'mystérieux' },
    'dim7': { notes: ['1', 'b3', 'b5', 'bb7'], intervals: ['1', 'b3', 'b5', 'bb7'], type: 'diminished', mood: 'mystérieux' },
    'm7b5': { notes: ['1', 'b3', 'b5', 'b7'], intervals: ['1', 'b3', 'b5', 'b7'], type: 'diminished', mood: 'mystérieux' },
    '9': { notes: ['1', '3', '5', 'b7', '9'], intervals: ['1', '3', '5', 'b7', '9'], type: 'extended', mood: 'joyeux' },
    'maj9': { notes: ['1', '3', '5', '7', '9'], intervals: ['1', '3', '5', '7', '9'], type: 'extended', mood: 'joyeux' },
    'm9': { notes: ['1', 'b3', '5', 'b7', '9'], intervals: ['1', 'b3', '5', 'b7', '9'], type: 'extended', mood: 'triste' },
    '11': { notes: ['1', '3', '5', 'b7', '9', '11'], intervals: ['1', '3', '5', 'b7', '9', '11'], type: 'extended', mood: 'neutre' },
    '13': { notes: ['1', '3', '5', 'b7', '9', '11', '13'], intervals: ['1', '3', '5', 'b7', '9', '11', '13'], type: 'extended', mood: 'joyeux' },
    'add9': { notes: ['1', '3', '5', '9'], intervals: ['1', '3', '5', '9'], type: 'extended', mood: 'joyeux' },
    '6': { notes: ['1', '3', '5', '6'], intervals: ['1', '3', '5', '6'], type: 'extended', mood: 'joyeux' },
    'm6': { notes: ['1', 'b3', '5', '6'], intervals: ['1', 'b3', '5', '6'], type: 'extended', mood: 'triste' },
}

const CHROMATIC_NOTES = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B']

const NOTE_NAMES: Record<string, string> = {
    'C': 'DO', 'C#': 'DO#', 'D': 'RE', 'D#': 'RE#', 'E': 'MI', 'F': 'FA',
    'F#': 'FA#', 'G': 'SOL', 'G#': 'SOL#', 'A': 'LA', 'A#': 'LA#', 'B': 'SI'
}

const INTERVAL_TO_SEMITONES: Record<string, number> = {
    '1': 0, 'b2': 1, '2': 2, 'b3': 3, '3': 4, '4': 5, 'b5': 6, '5': 7,
    'b6': 8, '6': 9, 'b7': 10, '7': 11, '9': 14, '11': 17, '13': 21, 'bb7': 9
}

// État réactif
const poly = new Tone.PolySynth({
    voice: Tone.Synth, // Synthé simple pour un son clair, type piano
    options: {
        oscillator: {
            type: "triangle" // son doux et classique
        },
        envelope: {
            attack: 0.005,   // très rapide comme un piano
            decay: 0.2,      // courte décroissance
            sustain: 0.4,    // sustain moyen
            release: 1,      // relâchement pour simuler le son d'une touche de piano
            attackCurve: "linear"
        }
    }
}).toDestination()



const isPlaying = ref(false)
const selectedRoot = ref('C')
const selectedChord = ref('')
const audioInitialized = ref(false)

const playMode = ref<PlayMode>({
    type: 'simultané',
    speed: 200
})

const filters = ref<Filters>({
    type: 'all',
    mood: 'all'
})

// Fonctions utilitaires
const getChordNotes = (root: string, chordType: string): string[] => {
    const chordData = CHORD_DEFINITIONS[chordType]
    if (!chordData) return [root]

    const rootIndex = CHROMATIC_NOTES.indexOf(root)
    return chordData.intervals.map(interval => {
        const semitones = INTERVAL_TO_SEMITONES[interval]
        const noteIndex = (rootIndex + semitones) % 12
        return CHROMATIC_NOTES[noteIndex]
    })
}

const getChordDisplayName = (root: string, chordType: string): string => {
    return `${NOTE_NAMES[root]}${chordType}`
}

const getMoodColor = (mood: string): string => {
    switch (mood) {
        case 'joyeux': return 'joyeux'
        case 'triste': return 'triste'
        case 'tendu': return 'tendu'
        case 'mystérieux': return 'mystérieux'
        default: return 'bg-gray-100 border-gray-300'
    }
}

// Initialisation de l'audio
const initializeAudio = async (): Promise<void> => {
    try {
        const context = Tone.getContext()
        if (context.state !== 'running') {
            await Tone.start()
            console.log('Contexte audio démarré')
        }

        // Création du PolySynth fonctionnel
        // const poly = new Tone.PolySynth(Tone.AMSynth).toDestination();

        // synth.value = new Tone.PolySynth(Tone.AMSynth).set({
        //     oscillator: { type: "fatsawtooth" },
        //     envelope: {
        //         attack: 0.01,
        //         decay: 0.1,
        //         sustain: 0.5,
        //         release: 0.4,
        //         attackCurve: "exponential"
        //     }
        // }).toDestination()

        audioInitialized.value = true
        console.log('Synthétiseur initialisé')
    } catch (error) {
        console.error('Erreur initialisation audio:', error)
    }
}

let timeouts: number[] = []
const clearAllTimeouts = () => {
    timeouts.forEach(id => clearTimeout(id))
    timeouts = []
}

// Fonction de lecture
const playChord = async (root: string, chordType: string): Promise<void> => {
    if (!poly) {
        console.error('Synthétiseur non disponible')
        return
    }

    // 🔇 Stoppe tout son en cours
    poly.releaseAll()
    isPlaying.value = false
    clearAllTimeouts()

    if (!audioInitialized.value) {
        await initializeAudio()
    }

    isPlaying.value = true
    const notes = getChordNotes(root, chordType)

    try {
        if (playMode.value.type === 'simultané') {
            const chordNotes = notes.map(note => `${note}4`)
            poly.triggerAttackRelease(chordNotes, 1) // 1 seconde
            timeouts.push(setTimeout(() => {
                isPlaying.value = false
            }, 1000) as unknown as number)
        } else {
            let playNotes = [...notes]

            if (playMode.value.type === 'arpège descendant') {
                playNotes.reverse()
            } else if (playMode.value.type === 'arpège alterné') {
                playNotes = [...notes, ...notes.slice().reverse().slice(1)]
            }

            playNotes.forEach((note, index) => {
                const timeoutId = setTimeout(() => {
                    // 🔇 stoppe avant de jouer la nouvelle note
                    poly.releaseAll()
                    poly.triggerAttackRelease(`${note}4`, 1) // 1 seconde max
                    if (index === playNotes.length - 1) {
                        timeouts.push(setTimeout(() => {
                            isPlaying.value = false
                        }, 1000) as unknown as number)
                    }
                }, index * playMode.value.speed)
                timeouts.push(timeoutId as unknown as number)
            })
        }
    } catch (error) {
        console.error('Erreur de lecture:', error)
        isPlaying.value = false
        clearAllTimeouts()
    }
}



const playNote = async (note: string): Promise<void> => {
    if (!audioInitialized.value) {
        await initializeAudio()
    }

    console.log(note)

    if (poly) {
        poly.triggerAttackRelease(note, '8n')
    } else {
        console.error('Synthétiseur non disponible après initialisation')
    }
}


// Propriétés calculées
const currentChordNotes = computed(() => {
    return getChordNotes(selectedRoot.value, selectedChord.value).map(note => `${note}4`)
})

const filteredChords = computed(() => {
    return Object.entries(CHORD_DEFINITIONS).filter(([chordType, data]) => {
        const typeMatch = filters.value.type === 'all' || data.type === filters.value.type
        const moodMatch = filters.value.mood === 'all' || data.mood === filters.value.mood
        return typeMatch && moodMatch
    })
})

// Gestionnaire de clic sur accord
const onChordClick = async (root: string, chordType: string): Promise<void> => {
    selectedChord.value = ""
    selectedRoot.value = ""


    selectedRoot.value = root
    selectedChord.value = chordType
    await playChord(root, chordType)
}

// Nettoyage
onUnmounted(() => {
    if (poly) {
        poly.dispose()
    }
})
</script>

<style>
.slider::-webkit-slider-thumb {
    appearance: none;
    width: 20px;
    height: 20px;
    background: #4f46e5;
    cursor: pointer;
    border-radius: 50%;
}

.slider::-moz-range-thumb {
    width: 20px;
    height: 20px;
    background: #4f46e5;
    cursor: pointer;
    border-radius: 50%;
    border: none;
}




/* Conteneur principal */
.daccords-div {
    min-height: 100vh;
    display: flex;
    justify-content: center;
}

.max-w-7xl {
    max-width: 1280px;
    width: 100%;
}

/* En-tête */
.text-center {
    text-align: center;
    margin-bottom: 32px;
}

h1 {
    font-size: 2.25rem;
    font-weight: bold;
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 12px;
    margin-bottom: 8px;
}

p {
    color: #4b5563;
    font-size: 1rem;
}

button {
    cursor: pointer;
    transition: all 0.2s;
    border: none;
}

button:disabled {
    cursor: not-allowed;
    opacity: 0.5;
}

/* Bouton activer audio */
.bg-green-600 {
    background-color: #16a34a;
    color: white;
    padding: 12px 24px;
    border-radius: 12px;
    font-weight: 600;
}

.bg-green-600:hover {
    background-color: #15803d;
}

/* Card / Panel */
.bg-white {
    background-color: white;
}

.rounded-xl {
    border-radius: 16px;
}

.shadow-lg {
    box-shadow: 0 10px 15px rgba(0, 0, 0, 0.1);
}

.p-6 {
    padding: 24px;
}

.mb-6 {
    margin-bottom: 24px;
}

.grid {
    display: grid;
    gap: 24px;
}

.grid-cols-1 {
    grid-template-columns: 1fr;
}

.md\:grid-cols-3 {
    grid-template-columns: repeat(3, 1fr);
}

/* Form inputs */
select,
input[type="range"] {
    width: 100%;
    padding: 12px;
    border: 1px solid #d1d5db;
    border-radius: 12px;
    font-size: 0.875rem;
}

input[type="range"] {
    height: 8px;
    background: #e5e7eb;
    appearance: none;
}

input[type="range"]::-webkit-slider-thumb {
    appearance: none;
    width: 20px;
    height: 20px;
    background: #4f46e5;
    cursor: pointer;
    border-radius: 50%;
}

input[type="range"]::-moz-range-thumb {
    width: 20px;
    height: 20px;
    background: #4f46e5;
    cursor: pointer;
    border-radius: 50%;
    border: none;
}

/* Tableau */
table {
    width: 100%;
    border-collapse: collapse;
}

thead {
    background-color: #f9fafb;
}

th,
td {
    border-right: 1px solid #e5e7eb;
    padding: 8px;
    font-weight: 500;
}

th {
    position: sticky;
    top: 0;
    background-color: #f9fafb;
    text-align: center;
}

td.sticky-left {
    position: sticky;
    left: 0;
    background-color: inherit;
}

td button {
    width: 100%;
    height: 48px;
    border-radius: 8px;
    font-size: 0.75rem;
    font-weight: 600;
    transition: all 0.2s;
}

td button:hover {
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    transform: scale(0.95);
}

td button.selected {
    box-shadow: 0 0 0 2px #6366f1;
    transform: scale(1.05);
}

/* Couleurs des humeurs */
.joyeux {
    background-color: #fef3c7;
    border: 2px solid #fde68a;
}

.triste {
    background-color: #dbeafe;
    border: 2px solid #93c5fd;
}

.tendu {
    background-color: #fee2e2;
    border: 2px solid #fca5a5;
}

.mysterieux {
    background-color: #ede9fe;
    border: 2px solid #c4b5fd;
}

.neutre {
    background-color: #f3f4f6;
    border: 2px solid #d1d5db;
}

/* Légende */
.legend {
    display: flex;
    flex-wrap: wrap;
    gap: 16px;
    margin-top: 24px;
}

.legend div {
    display: flex;
    align-items: center;
    gap: 8px;
}

.legend div span {
    font-size: 0.875rem;
    color: #374151;
}


.piano-div {
    position: sticky;
    top: 0;
    z-index: 10;

    background-color: white;
    border-radius: 8px;
    margin-bottom: 24px;
}


.btn {
    color: white;
    padding: 12px 24px;
    border-radius: 12px;
    font-weight: 600;
    font-size: 1rem;
}


.btn-activation-audio {
    background-color: #10b981;

}

.btn-activation-audio:hover {
    background-color: #059669;
}
</style>