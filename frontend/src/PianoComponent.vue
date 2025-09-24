<template>
    <div class="piano-component-div" ref="pianoContainer">
    </div>
</template>

<script setup lang="ts">
import { computed, ref, onMounted, onUnmounted, watch } from 'vue'
import { Instrument } from "piano-chart";

const pianoContainer = ref<HTMLDivElement | null>(null);
const piano = ref<Instrument>();

// Props
interface Props {
    highlightedNotes?: string[]
}

const props = withDefaults(defineProps<Props>(), {
    highlightedNotes: () => []
})



onMounted(() => {
    if (!pianoContainer.value) return;

    // Valeur de la première et dernière octave en fonction de si on est sur tel ou sur ordi
    const isMobile = window.outerWidth < 768;
    const startOctave = isMobile ? 3 : 2;
    const endOctave = isMobile ? 5 : 6;

    piano.value = new Instrument(pianoContainer.value, {
        startOctave,
        endOctave,
        showOctaveNumbers: true,
        highlightColor: '#3490dc',
    });


    piano.value.addKeyMouseDownListener((note) => {
        startAnimation([`${note.note}${note.accidental || ''}${note.octave}`]);
        emit('note-click', `${note.note}${note.accidental || ''}${note.octave}`);
    });

    piano.value.addKeyMouseUpListener((note) => {
        startAnimation([`${note.note}${note.accidental || ''}${note.octave}`]);
        emit('note-click', `${note.note}${note.accidental || ''}${note.octave}`);
    });


    piano.value.create();
});


watch(() => props.highlightedNotes, (newNotes) => {
    startAnimation(newNotes);
}, { immediate: true })


// Animation des notes
function startAnimation(notes: string[]) {

    if (!piano.value) return;

    piano.value.reload();

    for (const note of notes) {
        piano.value.keyDown(note);
    }

    setTimeout(() => {
        for (const note of notes) {
            piano.value?.keyUp(note);
        }
    }, 1000);

}

// Emits
const emit = defineEmits<{
    'note-click': [note: string]
}>()


// Nettoyage
onUnmounted(() => {
    if (piano.value) {
        piano.value.removeKeyMouseDownListener();
        piano.value.removeKeyMouseUpListener();
        piano.value.destroy();
    }

})

</script>

<style>
/* Styles spécifiques au composant Piano si nécessaire */
.piano-component-div {
    display: flex;
    justify-content: center;
    height: max-content;
    width: 100%;


    div {
        height: 100% !important;
        width: 100% !important;

    }
}
</style>