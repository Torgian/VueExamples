<template lang="pug">
v-row
  v-col.col-md-4.desktop-stepper
    v-card.text-center.d-flex.flex-column.step-container(flat)
      .logo-container.mx-auto.text-center
        img(
          :src="require(`~/assets/logo_green_e.svg`)"
          style={ width: "150px" }
        )
        div
          span(font-weight-bold) Epigno
        h1 アンケート
      v-card#stepperParent.steps(flat)
        v-stepper(v-model="currentStep" vertical alt-labels)
          template(v-for="q in questions")
            v-stepper-step.ml-5.mr-5.caption(
              :key="`${q.id}-step`"
              :complete="currentStep > q.id"
              :step="q.id"
              :id="`qstep-${q.id}`"
            ) {{ q.questionText }}
  v-col.col-xs-12.col-md-8.justify-center.pb-10
    .step-container.mobile-progress
      v-progress-linear(v-model="currentProgress" color="amber" height="25")
      template
        strong {{ currentStep }} / {{ questions.length }}
    template(v-for="q in questions")
      v-card.mx-auto.pt-3(flat v-if="currentStep === q.id && !nomination")
        v-card-title.headline.prime-question {{ q.questionText }}
        v-row.mb-5.mt-5.mr-0.ml-0(dense)
          v-card.col-6(flat)
            v-card-text.sub-question-neg {{ q.minusAnswer }}

          v-card.col-6(flat)
            v-card-text.sub-question-pos {{ q.plusAnswer }}
        v-row.mx-4
          v-card(flat color="transparent" width="100%")
            v-card-text.slider-area
              img.rating-image(
                :src="require('~/assets/survey-line-graph.svg')"
              )
              v-slider.pl-1.pr-1(
                v-model="q.rating"
                :rules="ratingRule"
                :min="0"
                :max="8"
                ticks
              )
        v-row
          v-col.text-center(xs-12)
            v-btn.mr-2.large-button(
              v-if="currentStep > 1"
              @click="onClickBack(q.id)"
              color="#5EB221"
              outlined
            ) 戻る
            v-btn.ml-2.large-button(
              v-if="finalQuestion === false"
              :disabled="q.rating == 4"
              @click="nextStep(q)"
            ) 次へ
            v-btn.ml-2.large-button(
              v-else
              :disabled="q.rating == 4"
              @click="isPublic ? submitAnswers() : (nomination = true)"
            ) {{ isPublic ? '回答を送信する' : '次へ' }}

    nomination-question(
      v-if="nomination"
      @set-nominated-user="nominatedUser = $event"
    )
      template(v-slot:actions)
        v-card-actions
          v-row.mt-5
            v-col.text-center
              v-btn.mr-2.large-button(
                @click="nomination = false"
                color="#5EB221"
                outlined
              ) 戻る
              v-btn.ml-2.large-button(@click="submitAnswers()") 回答を送信する

    template
      v-row(justify="center")
        v-dialog(v-model="dialog" persistent max-width="400")
          v-card.text-center
            template(v-if="dialogType === 'sendAnswers'")
              v-card-title.headline.justify-center アンケート回答を送信する
              v-card-text 回答を確定しますか？
              v-card-actions
                v-spacer
                v-btn(
                  @click="dialog = false"
                  :disabled="loading"
                  color="grey"
                  outlined
                ) キャンセル
                v-btn(
                  @click="confirmAnswers"
                  :loading="loading"
                  :disabled="loading"
                  color="primary"
                  depressed
                ) 確定する

            template(v-if="dialogType === 'error'")
              v-card-title.headline.justify-center エラー
              v-card-text アンケート送信中にエラーが発生しました。申し訳ありません。
              v-card-actions
                v-spacer
                v-btn(@click="dialog = false" color="primary" depressed) Ok
</template>
<script>
import PageMixin from '~/mixins/page.js'
import NominationQuestion from '~/pagePartials/survey/NominationQuestion.vue'

export default {
  name: 'SurveyQuestions',

  components: {
    NominationQuestion,
  },

  mixins: [PageMixin],

  props: {
    questions: {
      type: Array,
      required: true,
    },
    survey: {
      type: Object,
      required: true,
    },
    progress: {
      type: Number,
      required: true,
    },
    stepper: {
      type: Number,
      required: true,
    },
    answererName: {
      type: String,
      required: true,
    },
    answererByoto: {
      type: Number,
      default: null,
    },
    answererUnit: {
      type: Number,
      default: null,
    },
    answererTeam: {
      type: Number,
      default: null,
    },
    answererOccupation: {
      type: Number,
      default: null,
    },
    isPublic: {
      type: Boolean,
      required: true,
    },
  },
  data() {
    const finalQuestion = false
    const dialog = false
    const dialogType = ''
    const ratingRule = [r => r !== 4 || 'ご回答ください']
    const currentStep = this.stepper
    const currentProgress = this.progress
    const loading = false
    const nomination = false
    const nominatedUser = null
    return {
      ratingRule,
      finalQuestion,
      dialog,
      dialogType,
      currentProgress,
      currentStep,
      loading,
      nomination,
      nominatedUser,
    }
  },

  methods: {
    onClickBack(stepNumber) {
      this.currentStep = stepNumber > 1 ? stepNumber - 1 : 1
      this.currentProgress = (this.currentStep / this.questions.length) * 100
      const parent = document.getElementById('stepperParent')
      const element = document.getElementById('qstep-' + stepNumber)
      const elementOffset = 250
      const elementPosition = element.offsetTop
      const offsetPosition = elementPosition - elementOffset
      parent.scrollTo({
        top: offsetPosition,
        behavior: 'smooth',
      })
      if (this.nomination) this.nomination = false
      if (this.finalQuestion) this.finalQuestion = false
    },
    nextStep(q) {
      this.currentStep = q.id + 1
      this.currentProgress = (this.currentStep / this.questions.length) * 100
      const parent = document.getElementById('stepperParent')
      const element = document.getElementById('qstep-' + q.id)
      const elementOffset = 45
      const elementPosition = element.offsetTop
      const offsetPosition = elementPosition - elementOffset
      parent.scrollTo({
        top: offsetPosition,
        behavior: 'smooth',
      })
      if (this.currentStep === this.questions.length) {
        this.finalQuestion = true
      }
    },
    submitAnswers() {
      this.dialogType = 'sendAnswers'
      this.dialog = true
    },
    confirmAnswers() {
      this.loading = true
      const answers = this.questions.map(function (q) {
        const score = q.rating < 4 ? q.rating + 1 : q.rating
        return { surveyQuestionId: q.id, score }
      })

      const surveyAnswers = {
        userId: this.isPublic ? null : this.$me.id,
        surveyId: this.survey.id,
        answererName: this.answererName,
        answererByotoId: this.answererByoto,
        answererUnitId: this.answererUnit,
        answererTeamId: this.answererTeam,
        answererOccupationId: this.answererOccupation,
        surveyAnswers: answers,
        nominatedUserId: this.nominatedUser?.id,
      }

      this.$axios
        .$post('/survey-submissions', surveyAnswers)
        .then(res => {
          if (this.isPublic === false) {
            this.$store
              .dispatch('survey/retrievePendingSurveys', { force: true })
              .then(() => {
                return this.$router.push({
                  name: 'survey-thankyou',
                  params: { finalScores: res.score },
                })
              })
          } else {
            return this.$router.push({
              name: 'survey-public-thankyou',
              params: { finalScores: res.score },
            })
          }
        })
        .catch(e => {
          this.dialogType = 'error'
          console.warn('Error: ', e)
        })
    },

    setNomination(nomination) {
      this.nominatedUser = nomination
    },
  },
}
</script>

<style lang="stylus" scoped>
:deep(.v-slider__thumb)
  width: 20px !important
  height: 20px !important
  left: -10px
:deep(.v-slider__thumb:before)
  top: -8.5px
  left: -8.5px

.rating-image
  padding-left: 10px
  padding-right: 10px
  margin-bottom: -20px
  width: inherit

  @media screen and (max-width: 600px)
    width: 100%

@media screen and (min-width: 1400px)
  .large-button
    width: 200px
    height: 50px !important
    font-size: 1.5em

  .sub-question-neg, .sub-question-pos, .prime-question
    font-size: 1.4em

.desktop-stepper

  @media screen and (max-width: 872px)
    display: none


.mobile-progress

  @media screen and (min-width: 873px)
    display: none


.prime-question
  height: 150px
  align-items: start

.sub-question-neg
  margin-top: 10px

  @media screen and (max-width: 568px)
    writing-mode: vertical-rl
    text-orientation: upright
    height: 14em
    text-align: initial
    display: flex
    align-items: flex-end

  @media screen and (min-width: 569px)
    height: 100px

.sub-question-pos
  margin-top: 10px

  @media screen and (max-width: 600px)
    writing-mode: vertical-rl
    text-orientation: upright
    height: 14em
    text-align: initial

  @media screen and (min-width: 569px)
    height: 100px


.step-container
  max-height: 100%
  overflow-y: scroll

  .logo-container
    color: #5eb221

  .steps
    overflow-y: scroll
    max-height: 450px

    h1
      margin: 10px 0

    .v-stepper
      box-shadow: none
      font-size: 1.4em

      .v-stepper__step
        padding-left: 0


      .v-stepper__step--active .v-stepper__step__step
        background-color: #5eb221 !important

      .v-stepper__step--complete .v-stepper__step__step
        background-color: #257adb !important

.contents-container
  background-color: #cec2c2
  height: 100%
  overflow-y: auto

  .v-card.theme--light
    background-color: transparent

.slider-area
  padding-right: 0px
  padding-left: 0px
  :deep(.v-messages)
    padding-left: 1em

.no-evaluators
  color #bbb
</style>
