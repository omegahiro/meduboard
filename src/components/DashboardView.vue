<script setup>
import { ref, onMounted } from 'vue'

const userId = ref('')
const groupedData = ref({})
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

const computeQuestionStats = (questions, history) => {
  const questionMap = questions.reduce((acc, q) => {
    acc[q['問題ID']] = { category: q['カテゴリ'] }
    return acc
  }, {})

  const questionCount = Object.keys(questionMap).reduce((acc, qid) => {
    acc[qid] = { correct: 0, incorrect: 0 }
    return acc
  }, {})

  history.forEach(entry => {
    const { questionId, result } = entry
    if (questionCount[questionId]) {
      if (result) questionCount[questionId].correct++
      else questionCount[questionId].incorrect++
    }
  })

  return Object.entries(questionCount).map(([questionId, counts]) => {
    const status =
      counts.correct === 0 && counts.incorrect === 0 ? 'unlearned' :
      counts.correct > 0 && counts.incorrect === 0 ? 'correct' :
      counts.correct > counts.incorrect ? 'reviewed' :
      'incorrect'

    return {
      questionId,
      correct: counts.correct,
      incorrect: counts.incorrect,
      status,
      groupKey: questionId.slice(0, 4),
      category: questionMap[questionId]?.category || ''
    }
  })
}

const groupByKey = (arr) => {
  const grouped = {}
  arr.forEach(item => {
    if (!grouped[item.groupKey]) grouped[item.groupKey] = []
    grouped[item.groupKey].push(item)
  })
  return Object.fromEntries(
    Object.entries(grouped).sort(([a], [b]) => a.localeCompare(b))
  )
}

onMounted(async () => {
  userId.value = extractUserIdFromHash()
  if (!userId.value) return

  const questions = await fetchSheetData('questions')
  const history = await fetchSheetData('history')

  const filtered = history.filter(e => e.userId === userId.value)
  const stats = computeQuestionStats(questions, filtered)
  groupedData.value = groupByKey(stats)
})
</script>

<template>
  <div v-if="Object.keys(groupedData).length > 0" class="group-container">
    <div v-for="(groupItems, groupKey) in groupedData" :key="groupKey" class="group-table">
      <table border="1" style="border-collapse: collapse; width: 120px;">
        <thead><tr><th>{{ groupKey }}</th></tr></thead>
        <tbody>
          <tr v-for="item in groupItems" :key="item.questionId" :style="{
            backgroundColor:
              item.status === 'correct' ? '#d4edda' :
              item.status === 'incorrect' ? '#f8d7da' :
              item.status === 'reviewed' ? '#fff3cd' : '#e2e3e5'
          }">
            <td>{{ item.questionId }}({{ item.category[0] }}) {{ item.correct }}/{{ item.incorrect }}</td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
  <div v-else><p>データを読み込み中...</p></div>
</template>

<style scoped>
.group-container {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-top: 10px;
}
</style>
