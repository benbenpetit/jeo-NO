<template>
  <!-- <LoadingQuiz /> -->
  <div class="quiz-background"></div>
  <QuizOverlay />

  <Header @onModalOpen="onModalOpen" @onModalClose="onModalClose" :background="false">
  </Header>

  <div class="c-quiz-wrapper" style="visibility: hidden">
    <div class="character-quiz">
      <model-viewer
        class="--3d-model"
        src="/models/AnimationsPourMenus.glb"
        poster="/img/quiz/poster-quiz.webp"
        ar-modes="webxr"
        shadow-intensity="0"
        autoplay
        :animation-name="animationToPlay"
        disable-zoom
        disable-tap
        disable-pan
        interaction-prompt="none"
        camera-orbit="-20deg 80deg 2m"
        camera-target="0m 1.1m 0m"
      >
        <div slot="progress-bar" style="visibility: none"></div>
      </model-viewer>
    </div>

    <div class="c-quiz">
      <div class="c-quiz-modals">
        <Modal
          imgSrc="/icon/mic.svg"
          class="--blue"
          v-if="showQuiz && !quizCompleted"
          :isAnim="false"
        >
          <template v-slot:upper-img>
            <img
              :src="getCurrentQuestion?.img"
              alt=""
              style="height: 200px, z-index::10"
            />
          </template>
          <template v-slot:title>Journaliste</template>
          <template v-slot:content>
            <img
              src="/icon/quote-open.svg"
              alt=""
              style="height: 20px; transform: translateY(-15px)"
            />
            <p>{{ getCurrentQuestion?.question }}</p>
            <img
              src="/icon/quote-close.svg"
              alt=""
              style="height: 20px; align-self: flex-end; transform: translateY(10px)"
            />
          </template>
          <template v-slot:buttons>
            <ButtonUI
              @click="handleQuizClick(index)"
              v-for="(option, index) in getCurrentQuestion?.options"
              :key="index"
              class="--white"
              :class="getOptionClasses(index)"
              :disabled="selectedAnswer != null"
              :sound="false"
              :isActive="false"
            >
              <template v-slot:label>
                {{ option }}
              </template>
            </ButtonUI>
          </template>
        </Modal>
        <div class="c-info" v-if="!showQuiz">
          <Modal imgSrc="/icon/info.svg" class="gsap-quiz-info --blue" :isAnim="false">
            <template v-slot:title>Informations</template>
            <template v-slot:content>
              <p>{{ getCurrentQuestion?.info }}</p>
            </template>
            <template v-slot:buttons>
              <ButtonUI imgSrc="/icon/go.svg" @click="nextQuestion" class="--no-hover">
                <template v-slot:label>SUIVANT</template>
              </ButtonUI>
            </template>
          </Modal>
        </div>
      </div>
    </div>
  </div>

  <section v-show="quizCompleted">
    <div class="quiz-end-brush-wrapper">
      <div class="quiz-end-brush">
        <img :src="score >= 2 ? bluebrush : redbrush" alt="" />
        <p>{{ score >= 2 ? 'Bravo !' : 'Oulah...' }}</p>
      </div>
    </div>
    <div class="quiz-end-animals">
      <img :src="score >= 2 ? renardHappy : renardAngry" alt="" class="renard" />
      <img :src="score >= 2 ? gazelleHappy : gazelleAngry" alt="" class="gazelle" />
      <img :src="score >= 2 ? shibaHappy : shibaAngry" alt="" class="shiba" />
    </div>
  </section>

  <section v-show="showRecap">
    <Modal imgSrc="/icon/mic.svg" class="--blue --modal-recap-quiz">
      <template v-slot:title>Journaliste</template>
      <template v-slot:content>
        <div class="text-wrapper">
          <img
            src="/icon/quote-open.svg"
            alt=""
            style="height: 20px; transform: translateY(-15px)"
          />
          <p>
            {{
              score >= 2
                ? `Quelle culture ! Vous avez correctement répondu à la majorité de mes questions ! Voilà un résumé de ce que vous avez appris :`
                : `Ce n'est pas encore ça... Vous avez mal répondu à la plupart de mes questions ! Voilà un résumé de ce que vous avez appris :`
            }}
          </p>
          <img
            src="/icon/quote-close.svg"
            alt=""
            style="height: 20px; align-self: flex-end; transform: translateY(10px)"
          />
        </div>

        <div class="recap-table-title">
          <p class="--question">Questions</p>
          <p class="--answer">Réponses correctes</p>
        </div>
        <div class="recap-table">
          <div
            class="--table"
            v-for="({ question, answer, isValid }, index) in recapTable"
          >
            <div class="--entry">
              <div class="--question">
                <img
                  :src="
                    isValid ? '/icon/quiz-coche-valid.svg' : '/icon/quiz-cross-wrong.svg'
                  "
                  alt=""
                  class="--icon"
                />
                <p class="--index">{{ index + 1 }}</p>
                <p class="--text">
                  {{ question }}
                </p>
              </div>
              <p class="--answer">{{ answer }}</p>
            </div>
          </div>
        </div>
      </template>
      <template v-slot:buttons>
        <ButtonUI @click="endQuiz" imgSrc="/icon/go.svg">
          <template v-slot:label style="width: 350px">VOIR MON CLASSEMENT</template>
        </ButtonUI>
      </template>
    </Modal>
  </section>
</template>

<script setup lang="ts">
import ButtonUI from '@/components/common/ButtonUI.vue'
import Modal from '@/components/common/Modal.vue'
import Header from '@/components/common/Header.vue'
import QuizOverlay from '@/components/modules/Quiz/QuizOverlay.vue'
import LoadingQuiz from '@/components/modules/Quiz/LoadingQuiz.vue'
import { IQuestion } from '@/core/types/IQuiz'
import { ref, computed, onMounted, watch } from 'vue'
import { QUESTIONS_DATA } from '@/data/constants'
import { gsap } from 'gsap'
import { useSportStore } from '@/core/store/sport'
import { IScore } from '@/core/types/IScore'
import { useScoreStore } from '@/core/store/score'
import arrayShuffle from 'array-shuffle'
import { Howl, Howler } from 'howler'
// import { ModelObj } from 'vue-3d-model'
import '@google/model-viewer'

onMounted(async () => {
  questions.value = arrayShuffle(QUESTIONS_DATA)
  questions.value.unshift({
    question: `Belle performance ! Quelle était cette première figure ?`,
    answer: 3,
    options: ['PigeonFlip', 'Ollie', 'Kickflip', 'FS 270 Boardslide'],
    selected: null,
    img: '/img/quiz/first-fig.webp',
    info: `Le FS 270 Boardslide est la figure de prédiléction de Yuto, le premier médaillé olympique aux JO de Tokyo 2020`,
  })
})

const emit = defineEmits(['onEnd'])
const { setCurrentScore } = useScoreStore()
const { setSportStep } = useSportStore()
const showQuiz = ref(true)
const selectedAnswer = ref<number | null>(null)
const questions = ref<IQuestion[]>([])
const currentQuestion = ref(0)
const questionIndex = ref(1)

interface QAType {
  question: string
  answer: string
  isValid: boolean
}

const recapTable = ref<QAType[]>([])

const bluebrush = '/img/brush-blue-bravo.webp'
const redbrush = '/img/brush-red-oulah.webp'
const renardHappy = '/img/renard-happy.webp'
const renardAngry = '/img/renard-angry.webp'
const gazelleHappy = '/img/gazelle-happy.webp'
const gazelleAngry = '/img/gazelle-angry.webp'
const shibaHappy = '/img/shiba-happy.webp'
const shibaAngry = '/img/shiba-angry.webp'

let score = 0
let questionAnswered = 0
let quizCompleted = ref(false)
let showRecap = ref(false)
let soundtrackFilter = ref({ frequency: 20000 })
let soundGsapTl = gsap.timeline({})

let animationToPlay = ref<string>('P_Interview_Discution')

const quizOverlaySound = new Howl({
  src: ['/sounds/ui-sounds/sweep-1.mp3'],
  volume: 0.3,
})

const quizSoundtrack = new Howl({
  src: ['/sounds/soundtracks/quiz-and-skatepark.mp3'],
  volume: 0.7,
  loop: true,
})

const quizCorrectSound = new Howl({
  src: ['/sounds/ui-sounds/quiz-correct.mp3'],
  volume: 0.2,
  rate: 0.9,
})

const quizWrongSound = new Howl({
  src: ['/sounds/ui-sounds/quiz-wrong.mp3'],
  volume: 0.3,
})

const quizBlablaSound = new Howl({
  src: ['/sounds/ui-sounds/blabla-quiz.mp3'],
  volume: 0.2,
  sprite: {
    blabla1: [0, 2100, false],
    blabla2: [2000, 2450, false],
    blabla3: [5200, 6000, false],
    blabla4: [7000, 8000, false],
  },
})

const onModalOpen = () => {
  soundGsapTl.to(soundtrackFilter.value, {
    frequency: 500,
    duration: 0.6,
    ease: 'expo.out',
  })
}

const onModalClose = () => {
  soundGsapTl.to(soundtrackFilter.value, {
    frequency: 20000,
    duration: 0.6,
    ease: 'expo.in',
  })
}

watch(soundtrackFilter.value, () => {
  // @ts-ignore
  quizSoundtrack.frequency(soundtrackFilter.value.frequency)
})

const quizOverlayTimeline = gsap.timeline({
  onComplete: function () {
    quizAnimation()
  },
})
const quizTimeline = gsap.timeline({})
const quizEndTimeline = gsap.timeline({})
const quizRecapTimeline = gsap.timeline({})

const getCurrentQuestion = computed(() => {
  const question: IQuestion = questions.value[currentQuestion.value]
  if (!question) return
  question.index = currentQuestion.value
  return question
})

const endQuiz = () => {
  setSportStep('skate', 2)
}

const handleQuizClick = (index: number) => {
  if (!getCurrentQuestion.value) return
  selectedAnswer.value = index
  if (selectedAnswer.value === getCurrentQuestion.value.answer) {
    score++
    quizCorrectSound.play()
    recapTable.value.push({
      question: getCurrentQuestion.value.question,
      answer: getCurrentQuestion.value.options[getCurrentQuestion.value.answer],
      isValid: true,
    })
    animationToPlay.value = 'P_Interview_Correct2'
  } else {
    quizWrongSound.play()
    recapTable.value.push({
      question: getCurrentQuestion.value.question,
      answer: getCurrentQuestion.value.options[getCurrentQuestion.value.answer],
      isValid: false,
    })
    animationToPlay.value = 'P_Interview_Faux'
  }

  setTimeout(displayInfo, 1000)
}

const getOptionClasses = (index: number) => {
  if (!getCurrentQuestion.value) return
  const isCorrect =
    selectedAnswer.value !== null && index === getCurrentQuestion.value.answer
  const isWrong =
    selectedAnswer.value === index &&
    selectedAnswer.value !== getCurrentQuestion.value.answer

  const classes: { [key: string]: boolean } = {
    '--correct': isCorrect,
    '--wrong': isWrong,
  }

  return Object.keys(classes)
    .filter((key) => classes[key])
    .join(' ')
}

const setNextQuestion = () => {
  showQuiz.value = true
  selectedAnswer.value = null
  if (questionAnswered < 2) {
    currentQuestion.value++
    questionAnswered++
    animationToPlay.value = 'P_Interview_Discution'
  } else {
    quizCompleted.value = true
    emit('onEnd')
    gotoEndQuiz()
  }
}

const quizOverlayAnimation = () => {
  quizOverlayTimeline.add(function () {
    quizOverlaySound.play()
  })

  quizOverlayTimeline.fromTo(
    '.intro-quiz-title img',
    {
      x: '-100%',
    },
    {
      x: '0%',
      duration: 0.4,
      ease: 'Power2.easeInOut',
    },
  )
  quizOverlayTimeline.fromTo(
    '.intro-quiz-title .text p',
    {
      x: '-300%',
      duration: 0.4,
      ease: 'Power2.easeInOut',
    },
    {
      x: '0%',
      duration: 0.4,
      ease: 'Power2.easeInOut',
    },
    '-=0.3',
  )

  quizOverlayTimeline.to(
    '.intro-quiz-title img',
    {
      x: '-100%',
      duration: 0.4,
      ease: 'Power2.easeInOut',
    },
    '+=2',
  )

  quizOverlayTimeline.add(function () {
    quizOverlaySound.rate(0.9)
    quizOverlaySound.play()
  })

  quizOverlayTimeline.to(
    '.intro-quiz-title .text p',
    {
      x: '-300%',
      duration: 0.4,
      ease: 'Power2.easeInOut',
    },
    '-=0.4',
  )

  quizOverlayTimeline.to('.intro-quiz-wrapper', {
    display: 'none',
    duration: 0,
  })
}

const quizAnimation = () => {
  quizTimeline.fromTo(
    '.c-quiz-wrapper',
    {
      visibility: 'hidden',
      y: '-100%',
    },
    {
      visibility: 'visible',
      y: '0%',
      duration: 0,
      ease: 'Power4.easeInOut',
    },
  )

  quizTimeline.from(
    '.character-quiz',
    {
      x: '-100%',
      duration: 0.4,
      ease: 'Power4.easeInOut',
    },
    '-=0.2',
  )

  quizTimeline.from(
    '.c-modal-wrapper',
    {
      x: '100%',
      duration: 0.4,
      ease: 'Power4.easeInOut',
    },
    '-=0.3',
  )
}

const displayInfo = () => {
  quizTimeline.to('.c-quiz-modals', {
    x: '120%',
    duration: 0.4,
    ease: 'Power4.easeInOut',
  })

  quizTimeline.add(function () {
    showQuiz.value = false
  })

  quizTimeline.to('.c-quiz-modals', {
    x: '0%',
    duration: 0.6,
    ease: 'Power4.easeInOut',
  })

  quizTimeline.add(function () {
    quizBlablaSound.play(`blabla${questionIndex.value}`)
    questionIndex.value += 1
  })
}

const nextQuestion = () => {
  quizTimeline.to('.c-quiz-modals', {
    x: '120%',
    duration: 0.4,
    ease: 'Power4.easeInOut',
  })

  quizTimeline.add(function () {
    setNextQuestion()
  })

  quizTimeline.to(
    '.c-quiz-modals',
    {
      x: '0%',
      duration: 0.8,
      ease: 'Power4.easeInOut',
    },
    '+=0.3',
  )
}

const gotoEndQuiz = () => {
  quizEndTimeline.add(function () {
    if (score >= 2) {
      animationToPlay.value = 'Backflip'
    } else {
      animationToPlay.value = 'Nope'
    }
  })

  quizEndTimeline.fromTo(
    '.quiz-end-brush',
    {
      x: '100%',
    },
    {
      x: '0%',
      duration: 0.3,
      ease: 'Power2.easeInOut',
    },
  )

  quizEndTimeline.fromTo(
    '.quiz-end-animals img',
    {
      x: '300%',
      scaleX: '-1',
      y: '150px',
    },
    {
      x: '0%',
      duration: 0.3,
      ease: 'Power2.easeInOut',
      stagger: 0.15,
    },
    '-=0.2',
  )

  quizEndTimeline.to(
    '.quiz-end-animals img',
    {
      x: '300%',
      y: '150px',
      scaleX: -1,
      stagger: -0.15,
      rotate: '0deg',
      duration: 0.3,
      ease: 'Power2.easeInOut',
    },
    '+=0.7',
  )

  quizEndTimeline.to(
    '.quiz-end-brush',
    {
      x: '100%',
      duration: 0.3,
      ease: 'Power2.easeInOut',
    },
    '-=0.2',
  )

  quizEndTimeline.to('.quiz-end-animals', {
    visibility: 'hidden',
    duration: 0,
  })

  quizEndTimeline.add(function () {
    const tempScore: IScore = { points: score, sportId: 'skateQuiz' }
    setCurrentScore(tempScore)
    showRecap.value = true
    gotoRecapQuiz()
  })
}

const gotoRecapQuiz = () => {
  quizRecapTimeline.add(function () {
    animationToPlay.value = 'P_Interview_Penseur'
  })

  quizRecapTimeline.fromTo(
    '.--modal-recap-quiz',
    {
      x: '100%',
    },
    { x: '0%', duration: 0.5, ease: 'Power4.easeInOut' },
  )

  quizRecapTimeline.add(function () {
    quizBlablaSound.play(`blabla4`)
  })
}

onMounted(() => {
  Howler.stop()
  quizSoundtrack.fade(0, 0.7, 1000)
  quizSoundtrack.play()
  // @ts-ignore
  quizSoundtrack.addFilter({
    filterType: 'lowpass',
    frequency: soundtrackFilter.value.frequency,
    Q: 6.0,
  })
  quizOverlayAnimation()
})
</script>
