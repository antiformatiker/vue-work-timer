<script setup lang="ts">
import { computed, onMounted, ref, watch, onUnmounted } from 'vue'

type BreakTime = { start: string; end: string }

function timeToDate(time: string): Date {
  const [h, m] = time.split(':')
  const d = new Date()
  d.setHours(Number(h))
  d.setMinutes(Number(m))
  d.setSeconds(0)

  return d
}

const progress = ref(0)
const startTime = ref('08:00')
const newBreakStart = ref('08:00')
const newBreakEnd = ref('08:00')
const startDate = computed(() => timeToDate(startTime.value))
const checkPause = ref(true)
const duration = ref(7.8)
const pause = ref(30)
const checkBreak = ref(false)
const error = ref('')
const breaks = ref<BreakTime[]>([])
const endDate = computed(() => {
  let breakDurationsInMs = 0
  let durationInSeconds = duration.value * 60 * 60

  if (checkPause.value) {
    durationInSeconds += pause.value * 60
  }

  if (checkBreak.value) {
    breakDurationsInMs = breaks.value
      .map((b) => {
        const startDate = timeToDate(b.start)
        const endDate = timeToDate(b.end)

        return endDate.getTime() - startDate.getTime()
      })
      .reduce((v, a) => v + a, 0)
  }

  return new Date(startDate.value.getTime() + durationInSeconds * 1000 + breakDurationsInMs)
})
let interval: number

function addBreak() {
  error.value = ''
  const breakStartDate = timeToDate(newBreakStart.value)
  const breakEndDate = timeToDate(newBreakEnd.value)

  if (breakEndDate.getTime() <= breakStartDate.getTime()) {
    error.value =
      'Das Ende der Unterbrechung muss nach dem Beginn der Unterbrechung stattgefunden haben.'
    return
  }

  if (breakStartDate.getTime() <= startDate.value.getTime()) {
    error.value = 'Die Unterbrechung muss nach dem Einstempeln passiert sein.'
    return
  }

  breaks.value.push({
    start: newBreakStart.value,
    end: newBreakEnd.value,
  })

  newBreakStart.value = startTime.value
  newBreakEnd.value = startTime.value
}

function removeBreak(br: BreakTime) {
  breaks.value = breaks.value.filter((b: BreakTime) => {
    return b.start !== br.start && b.end !== br.end
  })
}

watch(duration, (newValue) => {
  if (newValue <= 6) {
    pause.value = 0
  } else if (newValue > 6 && newValue <= 9) {
    pause.value = 30
  } else if (newValue > 9) {
    pause.value = 45
  }
})

onMounted(() => {
  interval = setInterval(() => {
    const total = endDate.value.getTime() - startDate.value.getTime()
    const done = new Date().getTime() - startDate.value.getTime()

    progress.value = (100 / total) * done
  }, 300)
})

onUnmounted(() => {
  clearInterval(interval)
})
</script>

<template>
  <div>
    <div>
      <label for="startTime">Beginn</label>
      <input type="time" id="startTime" v-model="startTime" />
    </div>
    <div>
      <label for="checkpause">Pause?</label>
      <input type="checkbox" id="checkpause" v-model="checkPause" />
    </div>
    <div v-if="checkPause">
      <label for="pause">Pause in Minuten</label>
      <input type="number" id="pause" v-model="pause" />
    </div>
    <div>
      <label for="duration">Arbeitszeit (Dezimal)</label>
      <input type="number" id="duration" v-model="duration" />
    </div>
    <div>
      <label for="checkbreak">Unterbrechungen?</label>
      <input type="checkbox" id="checkbreak" v-model="checkBreak" />
    </div>
    <template v-if="checkBreak">
      <div v-for="(b, index) in breaks" :key="index">
        <div>
          <label for="break-start-{{ index }}">Von</label>
          <input readonly type="time" id="break-start-{{ index }}" :value="b.start" />
        </div>
        <div>
          <label for="break-end-{{ index }}">Bis</label>
          <input readonly type="time" id="break-end-{{ index }}" :value="b.end" />
        </div>
        <div>
          <button @click="removeBreak(b)">Entfernen</button>
        </div>
      </div>
      <div>
        <div>
          <label for="break-start">Von</label>
          <input type="time" id="break-start" v-model="newBreakStart" />
        </div>
        <div>
          <label for="break-end">Bis</label>
          <input type="time" id="break-end" v-model="newBreakEnd" />
        </div>
        <button @click="addBreak">Hinzufügen</button>
      </div>
    </template>
    <div>
      Du musst um <strong>{{ endDate.toLocaleTimeString() }}</strong> Feierabend machen um
      {{ duration }} Stunden <template v-if="checkPause">(Inkl. Pause)</template> zu erreichen.
    </div>
    <template v-if="error">
      <div style="color: red">{{ error }}</div>
    </template>
    <div>
      So viel hast du heute schon geschafft: <strong>{{ progress.toFixed(2) }}%</strong>
    </div>
  </div>
</template>
