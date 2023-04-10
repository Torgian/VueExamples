<template lang="pug">
v-select(
  clearable
  dense
  multiple
  counter
  chips
  :loading="filtersLoading"
  :disabled="filtersLoading"
  item-value="id"
  item-text="name"
  v-bind="$attrs"
  v-on="$listeners"
)
  template(v-slot:selection="{ item, index }")
    v-chip(v-if="index === 0")
      span {{ itemName(item.name) }}
    span.grey--text.caption(v-if="index === 1")
      | (+{{ valueLength - 1 }})
</template>
<script>
import { abbreviations } from '~/utilities/occupations'

export default {
  name: 'PersonnelFilter',
  props: {
    filtersLoading: {
      type: Boolean,
      required: true,
    },

    isOccupation: {
      type: Boolean,
      default: false,
    },
  },

  computed: {
    valueLength() {
      if (this.$attrs.value) {
        return this.$attrs.value.length
      } else {
        return 0
      }
    },
  },

  methods: {
    itemName(name) {
      return this.isOccupation ? abbreviations(name, 'roster') : name
    },
  },
}
</script>
