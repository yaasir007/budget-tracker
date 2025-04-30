<script setup>
import { ref, computed, onMounted, watch } from 'vue';

// State
const entries = ref([]);
const currentDate = ref(new Date());
const newEntry = ref({
  description: '',
  amount: '',
  type: 'expense',
  date: new Date().toISOString()
});

// Computed properties
const currentMonthName = computed(() => {
  return new Date(
    currentDate.value.getFullYear(),
    currentDate.value.getMonth()
  ).toLocaleString('default', { month: 'long' });
});

const currentYear = computed(() => {
  return currentDate.value.getFullYear();
});

const currentMonthKey = computed(() => {
  return `${currentDate.value.getFullYear()}-${currentDate.value.getMonth() + 1}`;
});

const filteredEntries = computed(() => {
  return entries.value.filter(entry => {
    const entryDate = new Date(entry.date);
    return entryDate.getMonth() === currentDate.value.getMonth() &&
           entryDate.getFullYear() === currentDate.value.getFullYear();
  });
});

const totalIncome = computed(() => {
  return filteredEntries.value
    .filter(entry => entry.type === 'income')
    .reduce((sum, entry) => sum + parseFloat(entry.amount), 0);
});

const totalExpenses = computed(() => {
  return filteredEntries.value
    .filter(entry => entry.type === 'expense')
    .reduce((sum, entry) => sum + parseFloat(entry.amount), 0);
});

const profit = computed(() => {
  return totalIncome.value - totalExpenses.value;
});

// Methods
function formatCurrency(value) {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD'
  }).format(value);
}

function formatDate(dateString) {
  const date = new Date(dateString);
  return date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'short',
    day: 'numeric'
  });
}

function addEntry() {
  if (!newEntry.value.description || !newEntry.value.amount) return;

  entries.value.push({
    description: newEntry.value.description,
    amount: parseFloat(newEntry.value.amount),
    type: newEntry.value.type,
    date: new Date().toISOString()
  });

  // Reset form
  newEntry.value = {
    description: '',
    amount: '',
    type: 'expense',
    date: new Date().toISOString()
  };

  // Save to storage
  saveEntries();
}

function deleteEntry(index) {
  const entryToDelete = filteredEntries.value[index];
  const globalIndex = entries.value.findIndex(entry =>
    entry.description === entryToDelete.description &&
    entry.date === entryToDelete.date &&
    entry.amount === entryToDelete.amount
  );

  if (globalIndex !== -1) {
    entries.value.splice(globalIndex, 1);
    saveEntries();
  }
}

function prevMonth() {
  const newDate = new Date(currentDate.value);
  newDate.setMonth(newDate.getMonth() - 1);
  currentDate.value = newDate;
}

function nextMonth() {
  const newDate = new Date(currentDate.value);
  newDate.setMonth(newDate.getMonth() + 1);
  currentDate.value = newDate;
}

function saveEntries() {
  localStorage.setItem('budget-tracker-entries', JSON.stringify(entries.value));
}

function loadEntries() {
  const savedEntries = localStorage.getItem('budget-tracker-entries');
  if (savedEntries) {
    entries.value = JSON.parse(savedEntries);
  }
}

// Lifecycle hooks
onMounted(() => {
  loadEntries();
});

// Register service worker for PWA
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js').then(registration => {
      console.log('ServiceWorker registration successful');
    }).catch(error => {
      console.log('ServiceWorker registration failed:', error);
    });
  });
}
</script>

<template>
  <div class="min-h-screen bg-gray-50 dark:bg-gray-900">
    <div class="container mx-auto px-4 py-8">
      <h1 class="text-2xl font-bold text-gray-900 dark:text-white mb-6">Monthly Budget Tracker</h1>

      <!-- Month Selector -->
      <div class="mb-6">
        <div class="flex items-center justify-between">
          <button @click="prevMonth" class="p-2 rounded-full hover:bg-gray-200 dark:hover:bg-gray-700">
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="text-gray-700 dark:text-gray-300"><path d="m15 18-6-6 6-6"></path></svg>
          </button>
          <h2 class="text-xl font-semibold text-gray-800 dark:text-gray-200">{{ currentMonthName }} {{ currentYear }}</h2>
          <button @click="nextMonth" class="p-2 rounded-full hover:bg-gray-200 dark:hover:bg-gray-700">
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="text-gray-700 dark:text-gray-300"><path d="m9 18 6-6-6-6"></path></svg>
          </button>
        </div>
      </div>

      <!-- Summary Cards -->
      <div class="grid grid-cols-1 md:grid-cols-3 gap-4 mb-6">
        <div class="bg-white dark:bg-gray-800 rounded-lg shadow p-4">
          <h3 class="text-sm font-medium text-gray-500 dark:text-gray-400">Income</h3>
          <p class="text-2xl font-bold text-green-600 dark:text-green-400">{{ formatCurrency(totalIncome) }}</p>
        </div>
        <div class="bg-white dark:bg-gray-800 rounded-lg shadow p-4">
          <h3 class="text-sm font-medium text-gray-500 dark:text-gray-400">Expenses</h3>
          <p class="text-2xl font-bold text-red-600 dark:text-red-400">{{ formatCurrency(totalExpenses) }}</p>
        </div>
        <div class="bg-white dark:bg-gray-800 rounded-lg shadow p-4">
          <h3 class="text-sm font-medium text-gray-500 dark:text-gray-400">Profit</h3>
          <p class="text-2xl font-bold" :class="profit >= 0 ? 'text-green-600 dark:text-green-400' : 'text-red-600 dark:text-red-400'">
            {{ formatCurrency(profit) }}
          </p>
        </div>
      </div>

      <!-- Add New Entry Form -->
      <div class="bg-white dark:bg-gray-800 rounded-lg shadow p-4 mb-6">
        <h3 class="text-lg font-medium text-gray-900 dark:text-white mb-4">Add New Entry</h3>
        <form @submit.prevent="addEntry" class="space-y-4">
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
              <label for="description" class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">Description</label>
              <input
                type="text"
                id="description"
                v-model="newEntry.description"
                required
                class="w-full px-3 py-2 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-2 focus:ring-primary-500 dark:bg-gray-700 dark:text-white"
                placeholder="Salary, Groceries, etc."
              >
            </div>
            <div>
              <label for="amount" class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">Amount</label>
              <input
                type="number"
                id="amount"
                v-model="newEntry.amount"
                required
                min="0.01"
                step="0.01"
                class="w-full px-3 py-2 border border-gray-300 dark:border-gray-600 rounded-md shadow-sm focus:outline-none focus:ring-2 focus:ring-primary-500 dark:bg-gray-700 dark:text-white"
                placeholder="0.00"
              >
            </div>
          </div>

          <div class="flex items-center space-x-4">
            <div class="flex items-center">
              <input
                type="radio"
                id="income"
                value="income"
                v-model="newEntry.type"
                class="h-4 w-4 text-primary-600 focus:ring-primary-500 border-gray-300"
              >
              <label for="income" class="ml-2 block text-sm text-gray-700 dark:text-gray-300">Income</label>
            </div>
            <div class="flex items-center">
              <input
                type="radio"
                id="expense"
                value="expense"
                v-model="newEntry.type"
                class="h-4 w-4 text-primary-600 focus:ring-primary-500 border-gray-300"
              >
              <label for="expense" class="ml-2 block text-sm text-gray-700 dark:text-gray-300">Expense</label>
            </div>
          </div>

          <div>
            <button
              type="submit"
              class="w-full flex justify-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-primary-600 hover:bg-primary-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-primary-500"
            >
              Add Entry
            </button>
          </div>
        </form>
      </div>

      <!-- Entries List -->
      <div class="bg-white dark:bg-gray-800 rounded-lg shadow overflow-hidden">
        <h3 class="text-lg font-medium text-gray-900 dark:text-white p-4 border-b border-gray-200 dark:border-gray-700">
          Entries
        </h3>

        <div class="divide-y divide-gray-200 dark:divide-gray-700">
          <div v-if="filteredEntries.length === 0" class="p-4 text-center text-gray-500 dark:text-gray-400">
            No entries for this month. Add your first income or expense!
          </div>

          <div v-for="(entry, index) in filteredEntries" :key="index" class="p-4 flex justify-between items-center">
            <div>
              <p class="text-sm font-medium text-gray-900 dark:text-white">{{ entry.description }}</p>
              <p class="text-xs text-gray-500 dark:text-gray-400">{{ formatDate(entry.date) }}</p>
            </div>
            <div class="flex items-center space-x-4">
              <span
                class="text-sm font-medium"
                :class="entry.type === 'income' ? 'text-green-600 dark:text-green-400' : 'text-red-600 dark:text-red-400'"
              >
                {{ entry.type === 'income' ? '+' : '-' }} {{ formatCurrency(entry.amount) }}
              </span>
              <button
                @click="deleteEntry(index)"
                class="text-gray-400 hover:text-red-500 dark:hover:text-red-400"
              >
                <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M3 6h18"></path><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"></path><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"></path></svg>
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
:root {
  --color-primary-500: #10b981;
  --color-primary-600: #059669;
  --color-primary-700: #047857;
}

.bg-primary-600 {
  background-color: var(--color-primary-600);
}

.hover\:bg-primary-700:hover {
  background-color: var(--color-primary-700);
}

.focus\:ring-primary-500:focus {
  --tw-ring-color: var(--color-primary-500);
}

.text-primary-600 {
  color: var(--color-primary-600);
}

@media (prefers-color-scheme: dark) {
  :root {
    --color-primary-500: #34d399;
    --color-primary-600: #10b981;
    --color-primary-700: #059669;
  }
}
</style>
