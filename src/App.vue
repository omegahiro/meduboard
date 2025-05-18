<script setup>
import { ref } from 'vue'
import DashboardView from './components/DashboardView.vue'
import ScoreView from './components/ScoreView.vue'

const activeTab = ref('dashboard')
</script>

<template>
  <div>
    <div style="margin-bottom: 16px;">
      <button @click="activeTab = 'dashboard'" :disabled="activeTab === 'dashboard'">進捗ダッシュボード</button> | 
      <button @click="activeTab = 'todayScore'" :disabled="activeTab === 'todayScore'">今日のスコア</button>
    </div>

    <DashboardView v-if="activeTab === 'dashboard'" />
    <ScoreView v-if="activeTab === 'todayScore'" />
  </div>
</template>
