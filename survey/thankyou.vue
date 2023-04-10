<template lang="pug">
thank-you(:isPublic="false" :scores="$root.context.params.finalScores")
</template>
<script>
import ThankYou from '~/pagePartials/survey/ThankYou.vue'

export default {
  components: {
    ThankYou,
  },
}
</script>
