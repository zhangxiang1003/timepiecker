<template>
  <div id="app">
    <time-period v-model="timeRange"></time-period>
  </div>
</template>

<script>
import TimePeriod from './components/TimePeriodPicker.vue'
export default {
  name: 'App',
  components: {
    TimePeriod
  },
  data() {
    return {
      timeRange: []
    }
  }
}
</script>
