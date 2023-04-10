<template lang="pug">
v-container
  div
    img(:src="require(`~/assets/logo_green_e.svg`)" style={ width: "150px" })
  h3 勤務アンケート No. {{ survey.number }}
  p {{ survey.facilityName }}
  h3
    p
      span 回答期限
      span.pl-3(style="font-weight: bold") {{ survey.expireAt }}
    p
      span 所要時間
      span.pl-6(style="font-weight: bold") およそ3分
  p {{ instructionText }}
  p.alert ※ アンケート回答途中で他画面へ遷移した場合、回答内容は保存されませんので、最後までご回答ください。
</template>

<script>
export default {
  name: 'SurveyIntro',
  props: {
    survey: {
      type: Object,
      required: true,
    },
    isPublic: {
      type: Boolean,
      required: true,
    },
    facilityType: {
      type: Number,
      required: true,
    },
  },

  computed: {
    instructionText() {
      return this.isPublic
        ? '下記情報を入力し、「次へ」を押してアンケートへお進みください。'
        : '「次へ」を押してアンケートへお進みください。'
    },
  },
}
</script>

<style scoped lang="stylus">
.alert
  font-size 80%
</style>
