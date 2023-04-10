<template lang="pug">
v-form
  v-container.name-area
    v-row
      v-col(cols="12" sm="12")
        v-text-field(
          v-model="name"
          :rules="[nameRules]"
          :change="$emit('enteredName', name)"
          label="名前"
          required
        )
    v-row
      v-col(cols="12" sm="12")
        v-select.mt-5(
          v-if="units.length"
          v-model="unit"
          :items="units"
          label="部署"
          item-text="name"
          item-value="id"
          :change="$emit('enteredUnit', unit)"
          dense
        )

        v-select.mt-5(
          v-if="occupations.length"
          v-model="occupation"
          :items="occupations"
          label="職種"
          item-text="name"
          item-value="id"
          :change="$emit('enteredOccupation', occupation)"
          dense
        )

        v-select.mt-5(
          v-if="byotos.length"
          v-model="byoto"
          :items="byotos"
          label="所属"
          item-text="name"
          item-value="id"
          :change="$emit('enteredByoto', byoto)"
          dense
        )

        expand(:value="showTeam")
          v-select.mt-5(
            v-model="team"
            :items="teams"
            label="チーム"
            item-value="id"
            :change="$emit('enteredTeam', team)"
            dense
          )
            template(v-slot:selection="{ item }")
              span.ml-2 {{ teamName(item) }}
            template(v-slot:item="{ item }")
              v-list-item-content
                v-list-item-title.caption.mb-2(
                  v-if="item.shiftGroupTitle"
                  v-html="item.shiftGroupTitle"
                )
                span.ml-2 {{ item.name }}
</template>
<script>
import cloneDeep from 'lodash/cloneDeep'

import Expand from '~/components/Containers/Expand.vue'

export default {
  name: 'NameQuery',

  components: {
    Expand,
  },

  props: {
    byotos: {
      type: Array,
      required: true,
    },

    facilityType: {
      type: Number,
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
    return {
      name: '',
      byoto: null,
      team: null,
      unit: null,
      occupation: null,
    }
  },
  computed: {
    nameRules() {
      return v => !!v || '名前必須'
    },

    /**
     * List of teams in the currently selected byoto
     */
    teams() {
      if (!this.byoto) {
        return []
      }

      const selectedByoto = this.byotos.find(b => b.id === this.byoto)
      const teams = cloneDeep(selectedByoto.teams)

      for (let i = 0; i < teams.length; i++) {
        if (i === 0) {
          teams[i].shiftGroupTitle = teams[i].shiftGroup.name
        } else {
          if (teams[i].shiftGroupId !== teams[i - 1].shiftGroupId) {
            teams[i].shiftGroupTitle = teams[i].shiftGroup.name
          }
        }
      }

      return teams
    },

    /**
     * Do we have any team in the currently selected byoto?
     * We exclude the isDontBelong team, which will always exist in any byoto.
     */
    anyTeam() {
      return this.teams.filter(t => !t.isDontBelong).length > 0
    },

    /**
     * Whether team input should be shown or not in the UI
     */
    showTeam() {
      return (
        this.facilityType !== this.$DEF('hospital.type.visit') &&
        this.byoto &&
        this.anyTeam
      )
    },
  },

  methods: {
    teamName(item) {
      if (item.shiftGroup.name) {
        return item.shiftGroup.name + 'ー' + item.name
      }
      return item.name
    },
  },
}
</script>
<style lang="stylus" scoped>
.name-area
  max-width: 425px
  padding-right: 0px
  padding-left: 0px
  padding-top: 0px
</style>
