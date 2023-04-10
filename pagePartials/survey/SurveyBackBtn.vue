<template lang="pug">
v-btn.backButton(@click="$emit('goBack')" small)
  v-icon {{ backButtonIcon }}
  span.ml-0 {{ selectedHeader }}
</template>
<script>
import { mdiArrowLeftBold } from '@mdi/js'

export default {
  name: 'SurveyBackBtn',

  props: {
    selectedHeader: {
      type: String,
      required: true,
    },
  },

  computed: {
    backButtonIcon() {
      return mdiArrowLeftBold
    },
  },
}
</script>
