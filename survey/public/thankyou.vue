<template lang="pug">
thank-you(:isPublic="true" :scores="$root.context.params.finalScores")
</template>
<script>
import ThankYou from '~/pagePartials/survey/ThankYou.vue'

export default {
  components: {
    ThankYou,
  },

  layout: 'public-survey',

  auth: false,
}
</script>
