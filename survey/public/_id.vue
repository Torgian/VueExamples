<template lang="pug">
h1.error-message(v-if="pageError") {{ pageError }}
v-stepper.mobilerate(v-else v-model="currentStep" horizontal alt-labels)
  v-stepper-step.no-show(step="1" complete)
  v-stepper-content.text-center(step="1")
    survey-intro.pb-0(
      :survey="survey"
      :isPublic="true"
      :facilityType="survey.facilityType"
    )

    name-query(
      @enteredName="setName"
      @enteredByoto="setByoto"
      @enteredTeam="setTeam"
      @enteredUnit="setUnit"
      @enteredOccupation="setOccupation"
      :byotos="survey.answererByotos"
      :facilityType="survey.facilityType"
      :occupations="survey.answererOccupations"
      :units="survey.answererUnits"
    )

    .text-center
      v-btn(
        :disabled="userName.length === 0"
        @click="currentStep = 2"
        depressed
      ) 次へ

  v-stepper-step.no-show(step="2" complete)
  v-stepper-content(step="2")
    survey-questions(
      :questions="questions"
      :survey="survey"
      :progress="progress"
      :stepper="stepper"
      :answererName="userName"
      :answererByoto="byoto"
      :answererUnit="unit"
      :answererTeam="team"
      :answererOccupation="occupation"
      :isPublic="true"
    )
</template>
<script>
import moment from 'moment'
import pageMixin from '~/mixins/page.js'
import NameQuery from '~/pagePartials/survey/NameQuery.vue'
import SurveyQuestions from '~/pagePartials/survey/SurveyQuestions.vue'
import SurveyIntro from '~/pagePartials/survey/SurveyIntro.vue'

export default {
  name: 'PublicSurveyPage',
  components: {
    NameQuery,
    SurveyQuestions,
    SurveyIntro,
  },
  mixins: [pageMixin],
  layout: 'public-survey',

  // Public page, so disable authentication
  auth: false,
  async asyncData({ $axios, params, error }) {
    let pageError = null
    let questions = null
    let survey = null
    await $axios
      .$get(`/surveys/public/${params.id}`, {
        params: {
          page: 'monolith',
        },
      })
      .then(res => {
        survey = res
        const today = moment().startOf('day')
        if (today < survey.distributeAt) {
          pageError =
            'アンケートはまだ配布されていません。管理者にお問い合わせください。'
        }
        if (today > survey.expireAt) {
          pageError = 'アンケートは終了しました。管理者にお問い合わせください。'
        }
      })
      .catch(e => {
        const err = {
          statusCode: e.response.status,
        }
        l(err)
        if (e.response.status === 404) {
          err.message = 'アンケートがありません。管理者にお問い合わせください。'
          err.customMessage = true
        }
        error(err)
      })

    await $axios
      .$get('/survey-questions', {
        params: {
          page: 'monolith',
        },
      })
      .then(res => {
        questions = res
        for (const q in questions) {
          questions[q].rating = 4
        }
      })

    const stepper = 1
    const progress = (stepper / questions.length) * 100

    return {
      questions,
      survey,
      progress,
      stepper,
      pageError,
    }
  },

  data() {
    const currentStep = 1
    const userName = ''
    const byoto = null
    const team = null
    const unit = null
    const occupation = null
    return {
      currentStep,
      userName,
      byoto,
      team,
      unit,
      occupation,
    }
  },
  methods: {
    setName(value) {
      this.userName = value
    },

    setByoto(value) {
      this.byoto = value
    },

    setTeam(value) {
      this.team = value
    },

    setUnit(value) {
      this.unit = value
    },

    setOccupation(value) {
      this.occupation = value
    },
  },
}
</script>
<style lang="stylus" scoped>
.no-show
  display: none
.error-message
  background: white
  padding: 30px
  text-align: center
  border-radius: 10px
@media screen and (max-width: 600px)
  .mobilerate
    padding-right: 5px
    padding-left: 5px
    :deep(.v-stepper__content)
      padding-right: 0px
      padding-left: 0px
</style>
