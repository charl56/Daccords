<template>
    <div class="daccords-div-main">
        <button v-if="!audioInitialized" @click="initializeAudio" class="btn-activation-audio">
            🔊 Activer l'audio
        </button>
        <div v-else class="daccords-div">
            <!-- Piano virtuel -->
            <div class="piano-div">
                <h2 class="text-xl font-semibold text-gray-800 mb-4">
                    Accord actuel: {{ getChordDisplayName(selectedRoot, selectedChord) }}
                </h2>
                <PianoComponent :highlighted-notes="currentChordNotes" @note-click="playNote" />
                <!-- Contrôles -->
                <div class="controles-div">
                    <!-- Mode de lecture -->
                    <div>
                        <label>Mode de lecture</label>
                        <select v-model="playMode.type">
                            <option value="simultané">Simultané</option>
                            <option value="arpège montant">Arpège montant</option>
                            <option value="arpège descendant">Arpège descendant</option>
                            <option value="arpège alterné">Arpège alterné</option>
                        </select>
                    </div>

                    <!-- Vitesse -->
                    <div>
                        <label>
                            Vitesse arpège: {{ playMode.speed }}ms
                        </label>
                        <input type="range" min="50" max="500" step="50" v-model.number="playMode.speed" />
                    </div>

                    <!-- Filtres -->
                    <div>
                        <label>Type</label>
                        <select v-model="filters.type">
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
                        <label>Humeur</label>
                        <select v-model="filters.mood">
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




            <!-- Tableau des accords -->
            <div class="tableau-accords-div">
                <div class="tableau-header">
                    <h2 class="tableau-title"> Tableau des accords ({{ filteredChords.length }} accords)
                    </h2>
                </div>

                <div class="tableau-container">
                    <table class="tableau">
                        <thead>
                            <tr>
                                <th class="col-note">Note</th>
                                <th v-for="[chordType] in filteredChords" :key="chordType" class="col-chord">
                                    {{ chordType || 'Maj' }}
                                </th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="(root, rowIndex) in CHROMATIC_NOTES" :key="root"
                                :class="rowIndex % 2 === 0 ? 'bg-gray-50' : 'bg-white'">
                                <td class="col-note">
                                    {{ NOTE_NAMES[root] }}
                                </td>
                                <td v-for="[chordType, chordData] in filteredChords" :key="`${root}-${chordType}`"
                                    class="col-cell">
                                    <button @click="onChordClick(root, chordType)" :disabled="!audioInitialized"
                                        class="w-full h-12 rounded-md text-xs font-semibold transition-all duration-200 border-2"
                                        :class="[
                                            selectedRoot === root && selectedChord === chordType
                                                ? 'chord-btn-selected'
                                                : '',
                                            !audioInitialized
                                                ? 'chord-btn-disabled'
                                                : 'chord-btn-hover',
                                            getMoodColor(chordData.mood)
                                        ]" :title="`${getChordDisplayName(root, chordType)} - ${chordData.mood}`">
                                        <div class="chord-btn-content">
                                            <span class="chord-label">{{ getChordDisplayName(root, chordType)
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
            <div class="legend-card">
                <h3 class="legend-title">Légende des humeurs</h3>
                <div class="legend-items">
                    <div class="legend-item">
                        <div class="legend-color joyeux"></div>
                        <span class="legend-label">Joyeux</span>
                    </div>
                    <div class="legend-item">
                        <div class="legend-color triste"></div>
                        <span class="legend-label">Triste</span>
                    </div>
                    <div class="legend-item">
                        <div class="legend-color tendu"></div>
                        <span class="legend-label">Tendu</span>
                    </div>
                    <div class="legend-item">
                        <div class="legend-color mysterieux"></div>
                        <span class="legend-label">Mystérieux</span>
                    </div>
                    <div class="legend-item">
                        <div class="legend-color neutre"></div>
                        <span class="legend-label">Neutre</span>
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
    '1': 0,
    'b2': 1, '2': 2, '#2': 3,
    'b3': 3, '3': 4, '#3': 5,
    '4': 5, '#4': 6,
    'b5': 6, '5': 7, '#5': 8,
    'b6': 8, '6': 9, '#6': 10,
    'bb7': 9, 'b7': 10, '7': 11,
    '9': 14, '#9': 15,
    '11': 17, '#11': 18,
    '13': 21, '#13': 22
}


// Synthétiseur polyphonique
const poly = new Tone.PolySynth(Tone.Synth).set({
    oscillator: {
        type: "fattriangle" // son doux et classique
    },
    envelope: {
        attack: 0.01,
        decay: 0.1,
        sustain: 0.5,
        release: 0.4,
        attackCurve: "exponential"
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

    // Permet de déclencher l'animation des touches
    selectedChord.value = ""
    selectedRoot.value = ""

    selectedRoot.value = root
    selectedChord.value = chordType


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
/* Conteneur du tableau */
.tableau-accords-div {
    background: #fff;
    border-radius: 0.75rem;
    /* rounded-xl */
    box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1),
        0 4px 6px -2px rgba(0, 0, 0, 0.05);
    /* shadow-lg */
    overflow: hidden;
}

/* En-tête du tableau */
.tableau-header {
    padding: 1rem;
    /* p-4 */
    background: #f9fafb;
    /* gray-50 */
    border-bottom: 1px solid #e5e7eb;
    /* border-b */
}

.tableau-title {
    font-size: 1.25rem;
    /* text-xl */
    font-weight: 600;
    /* font-semibold */
    color: #1f2937;
    /* gray-800 */
}

/* Container scroll horizontal */
.tableau-container {
    overflow-x: auto;
}

/* Table */
.tableau {
    width: 100%;
    border-collapse: collapse;
}

.tableau thead {
    background: #f9fafb;
    /* gray-50 */
}

.tableau th,
.tableau td {
    border-right: 1px solid #e5e7eb;
    /* border-r */
}

.col-note {
    position: sticky;
    left: 0;
    background: inherit;
    padding: 0.75rem 1rem;
    /* px-4 py-3 */
    text-align: left;
    font-weight: 600;
    color: #374151;
    /* gray-700 */
    z-index: 5;
}

.col-chord {
    min-width: 80px;
    /* min-w-[80px] */
    padding: 0.75rem 0.75rem;
    /* px-3 py-3 */
    text-align: center;
    font-weight: 600;
    color: #374151;
    /* gray-700 */
}

.col-cell {

    padding: 0.25rem;
    /* px-1 py-1 */
}

/* Alternance des lignes */
.row-even {
    background: #f9fafb;
    /* gray-50 */
}

.row-odd {
    background: #fff;
}

/* Bouton d’accord */
.chord-btn {
    width: 100% !important;
    height: 3rem;
    /* h-12 */
    border-radius: 0.375rem;
    /* rounded-md */
    font-size: 0.75rem;
    /* text-xs */
    font-weight: 600;
    /* font-semibold */
    border: 2px solid transparent;
    transition: all 0.2s ease;
    background: white;
    cursor: pointer;
}

/* Bouton sélectionné */
.chord-btn-selected {
    box-shadow: 0 0 0 2px #6366f1;
    /* ring-indigo-500 */
    transform: scale(1.05);
}

/* Bouton disabled */
.chord-btn-disabled {
    opacity: 0.5;
    cursor: not-allowed;
    transform: none;
}

/* Bouton hover */
.chord-btn-hover:hover {
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.chord-btn-hover:active {
    transform: scale(0.95);
}

/* Contenu du bouton */
.chord-btn-content {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    height: 100%;
}

.chord-label {
    color: #1f2937;
    /* gray-800 */
}


/* Conteneur principal */
.daccords-div-main {
    min-height: 100vh;
    width: 100%;

    display: flex;
    justify-content: center;
    align-items: center;

}


.daccords-div {
    width: 100%;

    h2 {
        margin: 0;
    }
}


.controles-div {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: space-evenly;
    padding: 5px 0px;
}

@media (max-width: 768px) {
    .controles-div {
        display: grid;
        grid-template-columns: 1fr 1fr;
        gap: 4px;
    }
}

.piano-div {
    position: sticky;
    top: 0;
    z-index: 10;
    background: linear-gradient(to bottom right, #eef2ff, #dbeafe);

    border-radius: 8px;
    margin-bottom: 24px;
}


.tableau-accords-div {
    width: 100%;
    overflow: auto;
}



.btn-activation-audio {
    width: 200px;
    background-color: #10b981;

}

.btn-activation-audio:hover {
    background-color: #059669;
}

.btn-play {
    color: black !important;
    transition: all 0.2s;
    width: 200px;
}


/* Card principale */
.legend-card {
    margin-top: 1.5rem;
    /* mt-6 */
    background: #fff;
    /* bg-white */
    border-radius: 0.75rem;
    /* rounded-xl */
    box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1),
        0 4px 6px -2px rgba(0, 0, 0, 0.05);
    /* shadow-lg */
    padding: 1rem;
    /* p-4 */
}

/* Titre */
.legend-title {
    font-size: 1.125rem;
    /* text-lg */
    font-weight: 600;
    /* font-semibold */
    color: #1f2937;
    /* gray-800 */
    margin-bottom: 0.75rem;
    /* mb-3 */
}

/* Container des items */
.legend-items {
    display: flex;
    flex-wrap: wrap;
    gap: 1rem;
    /* gap-4 */
}

/* Item individuel */
.legend-item {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    /* gap-2 */
}

/* Petite couleur */
.legend-color {
    width: 1rem;
    /* w-4 */
    height: 1rem;
    /* h-4 */
    border-radius: 0.25rem;
    /* rounded */
}

/* Label */
.legend-label {
    font-size: 0.875rem;
    /* text-sm */
    color: #374151;
    /* gray-700 */
}
</style>