<template lang="pug">
.page-questions
  side-nav-menu(menu="survey")

  v-stepper(v-model="mainScreenStepper")
    v-stepper-step.no-show(:complete="mainScreenStepper > 1" step="1")
    v-stepper-content.text-center(step="1")
      survey-intro(
        :survey="survey"
        :isPublic="false"
        :facilityType="$me.hospital.type"
      )
      .text-center
        v-btn(@click="mainScreenStepper = 2" depressed) 次へ

    v-stepper-step.no-show(step="2")
    v-stepper-content.mobilerate(step="2")
      survey-questions(
        :questions="questions"
        :survey="survey"
        :progress="progress"
        :stepper="stepper"
        :answererByoto="$store.state.auth.user.byoto.id"
        :answererName="$store.state.auth.user.name"
        :isPublic="false"
      )
</template>

<script>
import SideNavMenu from '~/components/Layouts/SideNavMenu/SideNavMenu.vue'

import SurveyQuestions from '~/pagePartials/survey/SurveyQuestions.vue'
import SurveyIntro from '~/pagePartials/survey/SurveyIntro.vue'

export default {
  name: 'Survey',
  components: {
    SideNavMenu,
    SurveyQuestions,
    SurveyIntro,
  },

  middleware: ['disableOnCorpUser'],

  async asyncData({ $axios, store, redirect }) {
    const [questions, survey] = await Promise.all([
      $axios.$get('/survey-questions', {
        params: {
          page: 'monolith',
        },
      }),
      store.dispatch('survey/retrievePendingSurveys').then(res => {
        return res[0]
      }),
    ])

    if (!survey) {
      return redirect('/survey/my-score')
    }

    for (const q in questions) {
      questions[q].rating = 4
    }
    const stepper = 1
    const progress = (stepper / questions.length) * 100
    return {
      questions,
      survey,
      progress,
      stepper,
    }
  },

  data() {
    const mainScreenStepper = 1
    return {
      mainScreenStepper,
    }
  },
}
</script>

<style lang="stylus" scoped>
body .container .grid-list-xl

  @media screen and (max-width: 600px)
    padding: 0px

.page-questions
  height: 100%
  .no-show
    display: none

  @media screen and (max-width: 600px)
    .mobilerate
      padding-right: 0px
      padding-left: 0px
</style>
