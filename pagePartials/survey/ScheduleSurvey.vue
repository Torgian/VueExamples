<template lang="pug">
v-card.mx-5(flat width="90%")
  v-form(v-model="formStatus.valid" @submit.prevent)
    v-card.card-container(flat width="100%")
      .d-flex.align-center.mr-2(v-if="!isRegularSurvey") 単回
        v-switch(
          v-model="autoSchedule"
          :disabled="isNotNew"
          @click="incrementKeys"
        )
        div 定期

      expand(:value="autoSchedule")
        v-row
          v-col.col-4
            v-select.monthly-selector(
              label="周期"
              suffix="ヶ月毎"
              dense
              v-model="monthlyPeriod"
              :items="monthRange"
            )

          v-col(v-if="isNotNew")
            v-btn.pr-1(outlined small @click="suspended = !suspended") {{ suspendLabel }}
              v-checkbox.suspend-checkbox(v-model="suspended" hide-details)

      v-menu(
        ref="dateStartMenu"
        v-model="dateStartMenu"
        :close-on-content-click="false"
        :return-value.sync="survey.distributeAt"
        transition="scale-transition"
        offset-y
        min-width="290px"
        :key="`dateStart${datePickerKey}`"
      )
        template(v-slot:activator="{ on, attrs }")
          expand(:value="expandMenu")
            v-text-field(
              v-model="survey.distributeAt"
              label="配布日"
              readonly
              v-bind="attrs"
              v-on="on"
              required
              :rules="[v => !!v || 'Required']"
            )
        v-date-picker(
          v-model="survey.distributeAt"
          :max="maximumDate"
          @change="$refs.dateStartMenu.save(survey.distributeAt)"
          no-title
          scrollable
          required
        )
      v-menu(
        ref="dateEndMenu"
        v-model="dateEndMenu"
        :close-on-content-click="false"
        :return-value.sync="survey.expireAt"
        transition="scale-transition"
        offset-y
        min-width="290px"
        :key="`dateEnd${datePickerKey}`"
      )
        template(v-slot:activator="{ on, attrs }")
          expand(:value="expandMenu")
            v-text-field(
              v-model="survey.expireAt"
              label="回答期限"
              readonly
              v-bind="attrs"
              v-on="on"
              required
              :rules="[v => !!v || 'Required']"
            )
        v-date-picker(
          v-model="survey.expireAt"
          :min="minimumDate"
          @change="$refs.dateEndMenu.save(survey.expireAt)"
          no-title
          scrollable
        )
      v-row.ml-0
        v-checkbox.mt-0(v-model="targetGlobal" label="全施設対象")

        v-checkbox.mt-0.ml-5(
          v-if="!autoSchedule"
          v-model="isPublic"
          label="公開"
        )

      v-select.target-selectors(
        v-model="targetUnitIds"
        :disabled="targetGlobal"
        :items="units"
        item-text="name"
        item-value="id"
        label="対象部署"
        multiple
        small-chips
        deletable-chips
      )

      v-select.target-selectors(
        v-model="targetByotoIds"
        :disabled="targetGlobal"
        :items="byotos"
        item-text="name"
        item-value="id"
        label="対象所属"
        multiple
        small-chips
        deletable-chips
      )

      v-select.target-selectors(
        v-model="targetOccupationIds"
        :disabled="targetGlobal"
        :items="occupations"
        item-text="name"
        item-value="id"
        label="対象職種"
        multiple
        small-chips
        deletable-chips
      )

      .submit-container.mb-5.mt-3.text-center
        v-btn.mr-2(@click="$emit('cancel')" small color="grey" outlined) キャンセル
        v-btn(
          :disabled="!formStatus.valid"
          :loading="formStatus.submitting"
          :dark="formStatus.valid"
          @click="submit"
          color="info"
          small
        )
          span(v-if="survey.id") 確定する
          span(v-else) 登録する
</template>
<script>
import moment from 'moment'
import Expand from '~/components/Containers/Expand.vue'

export default {
  name: 'ScheduleSurvey',

  components: {
    Expand,
  },
  props: {
    survey: {
      type: Object,
      required: true,
    },
    byotos: {
      type: Array,
      required: true,
    },
    units: {
      type: Array,
      required: true,
    },
    occupations: {
      type: Array,
      required: true,
    },
  },

  data() {
    const autoSchedule = !!this.survey.periodicSurvey
    const expandMenu = !this.survey.periodicSurvey
    const dateStartMenu = false
    const dateEndMenu = false
    const formStatus = { valid: false, submitting: false }
    const monthlyPeriod = this.survey.periodicSurvey
      ? this.survey.periodicSurvey.monthlyPeriod
      : 1
    const suspended = this.survey.periodicSurvey?.suspended ?? false

    const targetGlobal = this.survey.targetGlobal
    const targetUnitIds = this.survey.targetUnits
      ? this.survey.targetUnits.map(u => u.id)
      : []
    const targetByotoIds = this.survey.targetByotos
      ? this.survey.targetByotos.map(b => b.id)
      : []
    const targetOccupationIds = this.survey.targetOccupations
      ? this.survey.targetOccupations.map(o => o.id)
      : []
    const isPublic = !!this.survey.publicId

    /**
     * Required due to how v-menu reacts to dynamic
     *  DOM elements.
     *
     * This keeps the v-menu component attached to
     *  the text-field for the datepicker. It is
     *  incremented when the autoSchedule switch
     *  is clicked
     */
    const datePickerKey = 0

    return {
      targetUnitIds,
      targetByotoIds,
      targetOccupationIds,
      isPublic,
      targetGlobal,
      autoSchedule,
      monthlyPeriod,
      suspended,
      expandMenu,
      dateStartMenu,
      dateEndMenu,
      formStatus,
      datePickerKey,
    }
  },

  computed: {
    isNotNew() {
      return !!this.survey.id
    },

    isRegularSurvey() {
      return !!this.survey.id && !this.periodicSurveyId
    },

    minimumDate() {
      return moment(this.survey.distributeAt)
        .add(1, 'days')
        .format('YYYY-MM-DD')
    },

    maximumDate() {
      return moment(this.survey.expireAt)
        .subtract(1, 'days')
        .format('YYYY-MM-DD')
    },

    monthRange() {
      return [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
    },

    suspendLabel() {
      return this.suspended ? '再開する' : '停止する'
    },

    targetEndpoint() {
      return this.autoSchedule ? '/periodic-surveys' : '/surveys'
    },

    periodicSurveyId() {
      return this.survey.periodicSurvey?.id
    },

    submissionData() {
      const items = {
        targetGlobal: this.targetGlobal,
        targetUnitIds: this.targetUnitIds,
        targetByotoIds: this.targetByotoIds,
        targetOccupationIds: this.targetOccupationIds,
      }

      if (this.autoSchedule) {
        items.suspended = this.suspended
        items.monthlyPeriod = this.monthlyPeriod
      } else {
        items.isPublic = this.isPublic
        items.distributeAt = this.survey.distributeAt
        items.expireAt = this.survey.expireAt
      }

      return items
    },
  },

  watch: {
    targetGlobal(val) {
      if (val) {
        this.targetUnitIds = []
        this.targetByotoIds = []
        this.targetOccupationIds = []
      }
    },

    autoSchedule(val) {
      this.expandMenu = !val
      this.isPublic = false
    },
  },

  methods: {
    async submit() {
      this.formStatus.submitting = true

      const surveySchedule = this.submissionData

      /**
       * If there is a periodicSurveyId, or a survey id, send an update request.
       *
       */
      const idToPass = this.periodicSurveyId || this.survey.id

      if (idToPass) {
        await this.updateSurvey(idToPass, surveySchedule, this.targetEndpoint)
        this.$emit('updated')
      } else {
        await this.createSurveySchedule(surveySchedule, this.targetEndpoint)
        this.$emit('registered-survey')
      }
    },

    createSurveySchedule(newSchedule, targetEndpoint) {
      return this.$axios.$post(targetEndpoint, newSchedule)
    },

    updateSurvey(surveyId, editedSurvey, targetEndpoint) {
      return this.$axios.$put(`${targetEndpoint}/${surveyId}`, editedSurvey)
    },

    incrementKeys() {
      this.datePickerKey += 1
    },
  },
}
</script>
<style lang="stylus" scoped>
h1
  font-size: 1.8em
  min-width: 0

::-webkit-scrollbar
    -webkit-appearance: none
    width: 10px

::-webkit-scrollbar-thumb
  border-radius: 4px
  background-color: rgba(10, 20, 30, .5)
  box-shadow: 0 0 1px rgba(255, 255, 255, .5)

.target-selectors
  max-width 100%

.suspend-checkbox
  margin 0px 0px 0px 5px
  padding-bottom 4px
  pointer-events none

.monthly-selector
  max-width 105px
</style>
