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

    piano.value = new Instrument(pianoContainer.value, {
        startOctave: 2,
        endOctave: 6,
        showOctaveNumbers: true,
        highlightColor: '#3490dc',
    });


    piano.value.addKeyMouseDownListener((note) => {
        console.log(note)
        startAnimation([`${note.note}${note.accidental || ''}${note.octave}`]);
        emit('note-click', `${note.note}${note.accidental || ''}${note.octave}`);
    });

    piano.value.addKeyMouseDownListener((note) => {
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

    console.log('highlightedNotes', notes);
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

<style scoped>
/* Styles spécifiques au composant Piano si nécessaire */
.piano-component-div {
    display: flex;
    justify-content: center;
    height: 250px;
    width: 100%;


    div {
        height: 100% !important;
        width: 100% !important;

    }
}
</style>