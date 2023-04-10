<template lang="pug">
div
</template>
<script>
export default {
  name: 'Observer',

  data() {
    return {
      observer: null,
    }
  },

  destroyed() {
    // Disconnect all observers so as to not waste memory
    this.observer.disconnect()
    this.observer = null
  },

  mounted() {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry && entry.isIntersecting) {
          this.$emit('intersect')
        }
      },
      { threshold: 1.0 }
    )

    observer.observe(this.$el)
    this.observer = observer
  },
}
</script>
