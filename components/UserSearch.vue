<template lang="pug">
.search-wrapper
  .chip-wrapper(v-if="!single")
    v-chip(
      x-small
      close
      @click:close="remove(item)"
      v-for="item in selected"
      :key="item.id"
    )
      v-img(:src="userAvatar(item)" width="20")
      div {{ item.name }}
  v-autocomplete.ma-1(
    v-model="selected"
    :items="entries"
    :loading="isLoading"
    :search-input.sync="search"
    :disabled="disabled"
    hide-selected
    :label="label"
    :clearable="displayInSearch"
    :error-messages="errorMessage"
    item-value="name"
    :filter="customFilter"
    :multiple="!single"
    :menu-props="{ maxWidth: '200px', width: '200px' }"
    cache-items
    :dense="dense"
    :hide-details="hideDetails"
    :rules="rules"
    :no-data-text="isLoading ? 'ロード中' : '該当者が見つかりません'"
    placeholder="検索"
    return-object
    @click:clear="clearSelected()"
    @input="submitSelections()"
  )
    //- empty to prevent selections from appearing in search bar
    template(v-slot:selection="data")
      v-chip(v-if="single && displayInSearch")
        v-img(:src="userAvatar(data.item)" width="30")
        span {{ data.item.name }}
    template(v-slot:item="data")
      span.mr-1: v-img(:src="userAvatar(data.item)" width="30")
      span {{ data.item.name }}

    template(v-slot:append-item)
      observer(@intersect="queryApi(search)")
</template>
<script>
import cloneDeep from 'lodash/cloneDeep'
import debounce from 'lodash/debounce'

import Observer from '~/components/Elements/Observer.vue'

export default {
  name: 'UserSearch',

  components: {
    Observer,
  },

  props: {
    currentlySelectedUsers: {
      type: [Array, Object],
      default: null,
    },

    single: {
      type: Boolean,
      default: false,
    },

    displayInSearch: {
      type: Boolean,
      default: false,
    },

    disabled: {
      type: Boolean,
      default: false,
    },

    label: {
      type: String,
      default: 'スタッフ',
    },

    filter: {
      type: Object,
      default: () => ({}),
    },

    dense: {
      type: Boolean,
      default: false,
    },

    hideDetails: {
      type: Boolean,
      default: false,
    },

    errorMessage: {
      type: String,
      default: null,
    },

    rules: {
      type: Array,
      default: () => [],
    },
  },

  data() {
    return {
      search: null,
      isLoading: false,
      selected: null,
      entries: [],
      currentPage: 0,
      lastPage: null,
    }
  },

  watch: {
    currentlySelectedUsers: {
      deep: true,
      immediate: true,
      handler() {
        this.selected = cloneDeep(this.currentlySelectedUsers)
      },
    },

    search: debounce(function (value, oldValue) {
      /**
       *  Reset the currentPage and lastPage
       *   whenever searching is done, since the
       *   hits and page numbers will change with
       *   each API call.
       */
      if (value !== oldValue) {
        this.currentPage = 0
        this.lastPage = null
      }

      this.queryApi(value)
    }, 1000),
  },

  methods: {
    async queryApi(searchValue) {
      if (this.currentPage === this.lastPage) {
        return
      }

      this.currentPage += 1
      this.isLoading = true

      const searchParams = {
        keywords: searchValue,
        ...this.filter,
      }

      await this.$axios
        .$get(`/users?page=${this.currentPage}`, { params: searchParams })
        .then(res => {
          this.lastPage = res.lastPage
          this.entries = res.items
        })
        .catch(() => {
          this.$toast.error('エラーがありました。管理者とお問い合わせください')
        })
        .finally(() => (this.isLoading = false))
    },

    customFilter(item, queryText) {
      const staffNumber = item.staffNumber
      const userName = item.name

      return staffNumber.includes(queryText) || userName.includes(queryText)
    },

    remove(item) {
      this.selected = this.selected.filter(s => s.id !== item.id)
      this.submitSelections()
    },

    clearSelected() {
      this.selected = null
    },

    submitSelections() {
      this.$emit('submit-selections', this.selected)
    },
  },
}
</script>
<style lang="stylus" scoped>
.search-wrapper
  background-color white
  margin 0px
  padding 5px

.chip-wrapper
  max-width 250px
</style>
