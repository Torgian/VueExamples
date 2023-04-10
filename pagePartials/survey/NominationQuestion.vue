<template lang="pug">
v-card.mx-auto.pt-3(flat style={ height: "100%" })
  v-row.justify-center
    v-col.text-h6.text-center(cols="12") 今あなたが所属している部署の中で，あなたが一緒に働きたい人，
      p 信頼・尊敬する人を選択下さい
    v-col.mt-5(cols="4")
      user-search(
        @submit-selections="setNomination"
        :currently-selected-users="[]"
        single
        :filter="{ availability: [$DEF('enroll.active'), $DEF('enroll.away')] }"
      )
  v-row.justify-center
    span.mt-2.no-nominated-users(v-if="!nominatedUser") なし
    template(v-else)
      .mt-5.text-center
        v-img(:src="userAvatar(nominatedUser)" width="100")
        div {{ nominatedUser.name }}
  slot(name="actions")
</template>

<script>
import UserSearch from '~/components/Elements/UserSearch.vue'

export default {
  name: 'NominationQuestion',

  components: {
    UserSearch,
  },

  data() {
    return {
      nominatedUser: null,
    }
  },

  methods: {
    setNomination(nomination) {
      this.nominatedUser = nomination
      this.$emit('set-nominated-user', nomination)
    },
  },
}
</script>
<style lang="stylus" scoped>
.no-nominated-users
  color #bbb
</style>
