<template>
    <div class="time-range-container">
      <div class="time-group">
        <label class="time-label" for="start-hh">Annotation Start Time</label>
        <div class="time-input">
          <input
            id="start-hh"
            type="number"
            min="0"
            max="23"
            placeholder="HH"
            v-model.number="start.hh"
            @change="emitStartTime"
            aria-label="Start Hours"
            class="time-part"
          />
          :
          <input
            type="number"
            min="0"
            max="59"
            placeholder="MM"
            v-model.number="start.mm"
            @change="emitStartTime"
            aria-label="Start Minutes"
            class="time-part"
          />
          :
          <input
            type="number"
            min="0"
            max="59"
            placeholder="SS"
            v-model.number="start.ss"
            @change="emitStartTime"
            aria-label="Start Seconds"
            class="time-part"
          />
        </div>
      </div>
      <div class="time-group">
        <label class="time-label" for="end-hh">Annotation End Time</label>
        <div class="time-input">
          <input
            id="end-hh"
            type="number"
            min="0"
            max="23"
            placeholder="HH"
            v-model.number="end.hh"
            @change="emitEndTime"
            aria-label="End Hours"
            class="time-part"
          />
          :
          <input
            type="number"
            min="0"
            max="59"
            placeholder="MM"
            v-model.number="end.mm"
            @change="emitEndTime"
            aria-label="End Minutes"
            class="time-part"
          />
          :
          <input
            type="number"
            min="0"
            max="59"
            placeholder="SS"
            v-model.number="end.ss"
            @change="emitEndTime"
            aria-label="End Seconds"
            class="time-part"
          />
        </div>
      </div>
    </div>
  </template>

<script>
  export default {
    name: 'timestamp-editor',
    props: {
        videoAnnotationStartTime: {
            type: Number,
            default: null
        },
        videoAnnotationEndTime: {
            type: Number,
            default: null
        }
    },
    data () {
      return {
        start: { hh: '', mm: '', ss: '' },
        end: { hh: '', mm: '', ss: '' }
      }
    },
    watch: {
        videoAnnotationStartTime: {
            immediate: true,
            handler (newVal) {
                if (typeof newVal === 'number' && !isNaN(newVal)) {
                    const hh = Math.floor(newVal / 3600)
                    const mm = Math.floor((newVal % 3600) / 60)
                    const ss = Math.floor(newVal % 60)
                    this.start.hh = hh
                    this.start.mm = mm
                    this.start.ss = ss
                }
            }
        },
        videoAnnotationEndTime: {
            immediate: true,
            handler (newVal) {
                if (typeof newVal === 'number' && !isNaN(newVal)) {
                    const hh = Math.floor(newVal / 3600)
                    const mm = Math.floor((newVal % 3600) / 60)
                    const ss = Math.floor(newVal % 60)
                    this.end.hh = hh
                    this.end.mm = mm
                    this.end.ss = ss
                }
            }
        }
    },
    methods: {
        convertTimeToSeconds (inputField) {
            // Ensure numbers and handle empty fields
            const hh = Number(inputField.hh) || 0
            const mm = Number(inputField.mm) || 0
            const ss = Number(inputField.ss) || 0
            return hh * 3600 + mm * 60 + ss
        },
        emitStartTime () {
            const totalSeconds = this.convertTimeToSeconds(this.start)
            this.$emit('update:videoAnnotationStartTime', totalSeconds)
        },
        emitEndTime () {
            const totalSeconds = this.convertTimeToSeconds(this.end)
            this.$emit('update:videoAnnotationEndTime', totalSeconds)
        }
  }

}
</script>
