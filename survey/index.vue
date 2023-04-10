<template lang="pug"></template>

<script>
import PageMixin from '~/mixins/page.js'

export default {
  name: 'SurveyTop',
  mixins: [PageMixin],

  /**
   * Redirect user to appropriate survey page depending on current state
   */
  middleware({ store, redirect, $ua }) {
    const isMobile = $ua.isFromSmartphone() || $ua.isFromMobilephone()

    if (store.getters['user/isCorpUser']) {
      return redirect('/')
    }

    if (store.getters['survey/hasSurvey']) {
      // 回答待ちのアンケートがあるなら、アンケート回答画面に飛ぶ
      return redirect('/survey/questions')

      // なければ、
    } else if (!isMobile && store.getters['user/isSuperAdmin']) {
      // 管理者の場合、アンケート配布画面に飛ぶ
      return redirect('/survey/schedule')

      // でなければ、
    } else {
      // アンケート「私のスコア」画面に飛ぶ
      return redirect('/survey/my-score')
    }
  },
}
</script>
