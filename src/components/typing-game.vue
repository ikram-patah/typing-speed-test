<script setup>
    import { onKeyStroke } from '@vueuse/core';
    import { reactive, computed, onMounted } from 'vue';

    const state = reactive({
        words: [],
        isLoading: false,
        hasStarted: false,
        timeStart: null,
        timeEnd: null,
        isError: false,
        posWord: 0,
        posLetter: 0,
    });

    const wpm = computed(() => {
        if (state.timeStart && state.timeEnd) {
            const numberOfWords = state.words.length;
            const timeElapsed =
                (state.timeEnd.getTime() - state.timeStart.getTime()) / 1000;
            return Math.ceil(numberOfWords / (timeElapsed / 60));
        }
        return null;
    });

    function classWord(wordIndex) {
        return {
            is__current: wordIndex === state.posWord,
        };
    }

    function classLetter(wordIndex, letterIndex) {
        const isCurrentLetter =
            wordIndex === state.posWord && letterIndex === state.posLetter;
        return {
            is__current: isCurrentLetter,
            is__wrong: isCurrentLetter && state.isError,
        };
    }

    function startTimer() {
        if (!state.hasStarted) {
            state.hasStarted = true;
            state.timeStart = new Date();
        }
    }

    function endTimer() {
        state.hasStarted = false;
        state.timeEnd = new Date();
    }

    function moveMarkerToNextLetter() {
        const currentWordLength = state.words[state.posWord].length;
        const isLastLetter = currentWordLength === state.posLetter + 1;
        const isLastWord =
            isLastLetter && state.words.length === state.posWord + 1;

        if (isLastWord) {
            endTimer();
            return;
        }
        if (isLastLetter) {
            state.posWord += 1;
            state.posLetter = 0;
        } else {
            state.posLetter += 1;
        }
    }

    function isCorrectKeyPressed(e) {
        const pressedKey = e.key;
        const currentLetter = state.words[state.posWord][state.posLetter];
        return pressedKey === currentLetter;
    }

    function onType(e) {
        state.isError = false;
        if (e.key === ' ') {
            e.preventDefault();
        }
        if (isCorrectKeyPressed(e)) {
            startTimer();
            moveMarkerToNextLetter();
        } else if (state.hasStarted) {
            state.isError = true;
        }
    }

    async function fetchData() {
        state.isLoading = true;
        try {
            const response = await fetch(
                `https://random-word-api.vercel.app/api?words=50`
            );
            const words = await response.json();
            state.words = words.map((word) => word.split('').map((l) => l));
        } catch (error) {
            // eslint-disable-next-line no-console
            console.log(error);
        } finally {
            state.isLoading = false;
        }
    }

    async function restart() {
        await fetchData();
        state.hasStarted = false;
        state.timeStart = null;
        state.timeEnd = null;
        state.isError = false;
        state.posWord = 0;
        state.posLetter = 0;
    }

    onKeyStroke(true, onType);

    onMounted(fetchData);
</script>

<template>
    <div class="container my-5">
        <div class="mb-5 text-center">
            <h2>What is your WPM?</h2>
            <div class="text-muted">
                Test your typing speed. Start typing to start the timer.
            </div>
        </div>

        <div v-if="state.isLoading" class="loading-bar"></div>
        <div v-else class="words">
            <span
                v-for="(w, indexW) in state.words"
                :key="`word-${indexW}`"
                class="word"
                :class="classWord(indexW)"
            >
                <span
                    v-for="(l, indexL) in w"
                    :key="`letter-${indexL}`"
                    class="letter"
                    :class="classLetter(indexW, indexL)"
                >
                    {{ l }}
                </span>
            </span>
            <div v-if="wpm" class="results text-center mt-5 py-4">
                <div class="text-muted">
                    Words per minute: {{ wpm }}
                    -
                    <a href="#" @click.prevent="restart">Start over</a>
                </div>
            </div>
        </div>
    </div>
</template>
