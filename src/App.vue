<script setup>
import { ref, onMounted } from 'vue'


// ユーザーIDをwindow.location.hashから取得
const userId = ref('')

// グループごとの問題データを保持
const groupedData = ref({})

// Base URL of the Google Apps Script backend
const url = import.meta.env.VITE_GAS_DB_URL

// ハッシュからuserIdを取得する関数
const extractUserIdFromHash = () => {
  const hash = window.location.hash
  const params = new URLSearchParams(hash.slice(1)) // #を除去してURLSearchParamsを使う
  return params.get('userId') || ''  // userIdパラメータを取得。なければ空文字
}

// Fetch a sheet from the GAS backend
const fetchSheetData = async (sheetName) => {
  const response = await fetch(`${url}?sheetName=${sheetName}`)
  if (!response.ok) {
    throw new Error(`Failed to fetch ${sheetName}`)
  }
  return await response.json()
}

// Compute stats per questionId from user's history
const computeQuestionStats = (questionIds, history) => {
  // Initialize stats object per questionId
  const questionCount = questionIds.reduce((acc, qid) => {
    acc[qid] = { correct: 0, incorrect: 0 }
    return acc
  }, {})

  // Accumulate correct/incorrect counts
  history.forEach(entry => {
    const { questionId, result } = entry
    if (questionCount[questionId]) {
      if (result) {
        questionCount[questionId].correct++
      } else {
        questionCount[questionId].incorrect++
      }
    }
  })

  // Convert to array with status and groupKey
  return Object.entries(questionCount).map(([questionId, counts]) => {
    const { correct, incorrect } = counts

    // Determine learning status
    const status =
      correct === 0 && incorrect === 0 ? 'unlearned' :
        correct > 0 && incorrect === 0 ? 'correct' :
          correct > incorrect ? 'reviewed' :
            'incorrect'

    return {
      questionId,
      correct,
      incorrect,
      status,
      groupKey: questionId.slice(0, 4)
    }
  })
}

// Group question stats by groupKey
const groupByKey = (questionStatsArray) => {
  const grouped = {}

  questionStatsArray.forEach(item => {
    const key = item.groupKey
    if (!grouped[key]) {
      grouped[key] = []
    }
    grouped[key].push(item)
  })

  // Sort groups by key
  return Object.fromEntries(
    Object.entries(grouped).sort(([a], [b]) => a.localeCompare(b))
  )
}

// Main async function for fetching and processing data
onMounted(async () => {
  userId.value = extractUserIdFromHash() // URLのハッシュからユーザーIDを取得

  if (!userId.value) {
    console.error('User ID is not specified in the URL hash.')
    return
  }

  try {
    // Fetch question list
    const questions = await fetchSheetData('questions')
    const questionIds = questions.map(q => q['問題ID'])

    // Fetch history list
    const history = await fetchSheetData('history')

    // Filter history for current user
    const filteredHistory = history.filter(entry => entry.userId === userId.value)

    // Compute question stats and group them
    const questionStats = computeQuestionStats(questionIds, filteredHistory)
    groupedData.value = groupByKey(questionStats)

    console.log('Grouped Data:', groupedData.value)
  } catch (error) {
    console.error('Error loading data:', error)
  }
})
</script>

<template>
  <!-- User ID input field (commented out, since user ID is from URL now) -->
  <!-- ユーザーID: <input v-model="userId" style="width: 300px;" /> -->

  <!-- Display data if available -->
  <div v-if="Object.keys(groupedData).length > 0" class="group-container">
    <div v-for="(groupItems, groupKey) in groupedData" :key="groupKey" class="group-table">
      <table border="1" style="border-collapse: collapse; width: 100px;">
        <thead>
          <tr>
            <th>{{ groupKey }}</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="item in groupItems" :key="item.questionId" :style="{
            backgroundColor:
              item.status === 'correct' ? '#d4edda' :
                item.status === 'incorrect' ? '#f8d7da' :
                  item.status === 'reviewed' ? '#fff3cd' :
                    '#e2e3e5'
          }">
            <td>
              <!-- Display questionId and correct/incorrect counts -->
              {{ item.questionId }} {{ item.correct }}/{{ item.incorrect }}
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>

  <!-- Show loading message when no data -->
  <div v-else>
    <p>データを読み込み中...</p>
  </div>
</template>

<style scoped>
/* Container for group tables (flex layout) */
.group-container {
  display: flex;
  /* flex-wrap: wrap; */
  gap: 10px;
  margin-top: 10px;
}

/* Individual group table as a card */
/* .group-table {
  flex: 0 0 calc((100% - (16px * 5)) / 6);
  box-sizing: border-box;
} */
</style>
