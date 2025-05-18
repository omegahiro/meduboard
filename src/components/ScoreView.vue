<script setup>
import { ref, onMounted } from 'vue'

const userId = ref('')
const todayStats = ref({ correct: 0, incorrect: 0, total: 0 })
const isLoading = ref(true) // 読み込み中フラグ
const url = import.meta.env.VITE_GAS_DB_URL

const extractUserIdFromHash = () => {
  const hash = window.location.hash
  const params = new URLSearchParams(hash.slice(1))
  return params.get('userId') || ''
}

const fetchSheetData = async (sheetName) => {
  const res = await fetch(`${url}?sheetName=${sheetName}`)
  if (!res.ok) throw new Error(`Failed to fetch ${sheetName}`)
  return await res.json()
}

const computeTodayStats = (history) => {
  const today = new Date().toISOString().slice(0, 10)
  let correct = 0, incorrect = 0

  history.forEach(entry => {
    const date = entry.timestamp?.slice(0, 10)
    if (date === today && entry.userId === userId.value) {
      entry.result ? correct++ : incorrect++
    }
  })

  return {
    correct,
    incorrect,
    total: correct + incorrect
  }
}

onMounted(async () => {
  try {
    userId.value = extractUserIdFromHash()
    if (!userId.value) return

    const history = await fetchSheetData('history')
    todayStats.value = computeTodayStats(history)
  } catch (error) {
    console.error('データの取得に失敗しました:', error)
  } finally {
    isLoading.value = false // 読み込み完了
  }
})
</script>

<template>
  <div>    
    <div v-if="isLoading">
      <p>データを読み込み中...</p>
    </div>

    <div v-else>
      <p>正解数: {{ todayStats.correct }}</p>
      <p>不正解数: {{ todayStats.incorrect }}</p>
      <p>合計: {{ todayStats.total }}</p>
      <p>正答率: <strong>{{ todayStats.total > 0 ? ((todayStats.correct / todayStats.total) * 100).toFixed(1) + '%' : '－' }}</strong></p>
    </div>
  </div>
</template>
