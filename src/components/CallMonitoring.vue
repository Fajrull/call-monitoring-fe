<script setup lang="ts">
import { ref, onMounted, watch } from 'vue';

interface CallRecord {
  callId: string;
  callTimestamp: string;
  csName: string;
  customerName: string;
  sentimentScore: number;
  createdAt: string;
}

interface Paging {
  totalPages: number;
  totalElements: number;
  page: number;
  size: number;
  hasNext: boolean;
  hasPrevious: boolean;
}

interface ApiResponse {
  statusCode: number;
  message: string;
  data: CallRecord[];
  paging: Paging;
}

const records = ref<CallRecord[]>([]);
const paging = ref<Paging>({
  totalPages: 0,
  totalElements: 0,
  page: 1, 
  size: 5,
  hasNext: false,
  hasPrevious: false,
});
const loading = ref(false);
const error = ref<string | null>(null);
const search = ref('');
const startDate = ref('');
const endDate = ref('');
const sentiment = ref('');
const sortColumn = ref('csName');
const sortDirection = ref('asc');

const today = new Date();
const threeMonthsAgo = new Date();
threeMonthsAgo.setMonth(today.getMonth() - 3);

const maxDate = today.toISOString().split('T')[0];
const minDate = threeMonthsAgo.toISOString().split('T')[0];
const buildUrl = () => {
  const url = new URL(import.meta.env.VITE_API_URL || '/api/v1/call-monitoring', window.location.origin);
  
  if (search.value) url.searchParams.append('search', search.value);
  url.searchParams.append('page', paging.value.page.toString());
  url.searchParams.append('size', paging.value.size.toString());
  if (sortColumn.value) url.searchParams.append('sort', sortColumn.value);
  if (sortDirection.value) url.searchParams.append('direction', sortDirection.value);
  if (startDate.value) url.searchParams.append('startDate', startDate.value);
  if (endDate.value) url.searchParams.append('endDate', endDate.value);
  if (sentiment.value) url.searchParams.append('sentiment', sentiment.value);

  return url.toString();
};

const fetchData = async () => {
  loading.value = true;
  error.value = null;
  
  try {
    const response = await fetch(buildUrl());
    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
    
    const result: ApiResponse = await response.json();
    records.value = result.data || [];
    paging.value = result.paging || paging.value;
  } catch (e: any) {
    error.value = e.message || 'Failed to fetch data';
    records.value = [];
  } finally {
    loading.value = false;
  }
};const handleSort = (column: string) => {
  if (sortColumn.value === column) {
    sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc';
  } else {
    sortColumn.value = column;
    sortDirection.value = 'asc';
  }
};

const changePage = (newPage: number) => {
  paging.value.page = newPage;
  fetchData();
};
const formatDate = (dateStr: string) => {
  if (!dateStr) return '-';
  const date = new Date(dateStr);
  return new Intl.DateTimeFormat('en-GB', {
    dateStyle: 'medium',
    timeStyle: 'short'
  }).format(date);
};

const getSentimentBadgeClass = (score: number) => {
  if (score >= 70) return 'badge-success';
  if (score >= 40) return 'badge-warning';
  return 'badge-danger';
};

watch([search, startDate, endDate, sentiment, sortColumn, sortDirection, () => paging.value.size], () => {
  paging.value.page = 1; 
  fetchData();
});



onMounted(() => {
  fetchData();
});
</script>

<template>
  <div class="card">
    <div class="filters-bar">
      <div class="filter-group">
        <label for="search">Search</label>
        <input 
          id="search" 
          type="text" 
          v-model.lazy="search" 
          placeholder="Keyword..." 
          class="input-field" 
        />
      </div>

      <div class="filter-group">
        <label for="startDate">Start Date</label>
        <input 
          id="startDate" 
          type="date" 
          v-model="startDate" 
          :min="minDate" 
          :max="maxDate"
          class="input-field" 
        />
      </div>

      <div class="filter-group">
        <label for="endDate">End Date</label>
        <input 
          id="endDate" 
          type="date" 
          v-model="endDate" 
          :min="startDate || minDate" 
          :max="maxDate"
          class="input-field" 
        />
      </div>

      <div class="filter-group">
        <label for="sentiment">Sentiment</label>
        <select id="sentiment" v-model="sentiment" class="input-field">
          <option value="">All</option>
          <option value="BELOW_70">Di bawah 70%</option>
          <option value="ABOVE_OR_EQUAL_70">70% atau lebih</option>
        </select>
      </div>
    </div>

    <div v-if="error" class="empty-state" style="color: var(--danger)">
      <p>Error: {{ error }}</p>
    </div>

    <div class="table-container">
      <table>
        <thead>
          <tr>
            <th>No.</th>
            <th @click="handleSort('callId')" class="sortable" :class="{ 'active-sort': sortColumn === 'callId' }">
              Call ID
              <span class="sort-icon">{{ sortColumn === 'callId' && sortDirection === 'desc' ? '▼' : '▲' }}</span>
            </th>
            <th @click="handleSort('callTimestamp')" class="sortable" :class="{ 'active-sort': sortColumn === 'callTimestamp' }">
              Call Timestamp
              <span class="sort-icon">{{ sortColumn === 'callTimestamp' && sortDirection === 'desc' ? '▼' : '▲' }}</span>
            </th>
            <th @click="handleSort('csName')" class="sortable" :class="{ 'active-sort': sortColumn === 'csName' }">
              CS Name
              <span class="sort-icon">{{ sortColumn === 'csName' && sortDirection === 'desc' ? '▼' : '▲' }}</span>
            </th>
            <th @click="handleSort('customerName')" class="sortable" :class="{ 'active-sort': sortColumn === 'customerName' }">
              Nama Nasabah
              <span class="sort-icon">{{ sortColumn === 'customerName' && sortDirection === 'desc' ? '▼' : '▲' }}</span>
            </th>
            <th @click="handleSort('sentimentScore')" class="sortable" :class="{ 'active-sort': sortColumn === 'sentimentScore' }">
              Sentiment Score
              <span class="sort-icon">{{ sortColumn === 'sentimentScore' && sortDirection === 'desc' ? '▼' : '▲' }}</span>
            </th>
          </tr>
        </thead>
        
        <tbody v-if="loading && records.length === 0">
          <tr>
            <td colspan="6" class="empty-state">
              <div class="spinner"></div>
              <p>Memuat data...</p>
            </td>
          </tr>
        </tbody>

        <tbody v-else-if="records.length === 0">
          <tr>
            <td colspan="6" class="empty-state">
              <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M20 13V6a2 2 0 00-2-2H6a2 2 0 00-2 2v7m16 0v5a2 2 0 01-2 2H6a2 2 0 01-2-2v-5m16 0h-2.586a1 1 0 00-.707.293l-2.414 2.414a1 1 0 01-.707.293h-3.172a1 1 0 01-.707-.293l-2.414-2.414A1 1 0 006.586 13H4" />
              </svg>
              <p>No records match the active criteria.</p>
            </td>
          </tr>
        </tbody>

        <tbody v-else>
          <tr v-for="(record, index) in records" :key="record.callId">
            <td>{{ (paging.page - 1) * paging.size + index + 1 }}</td>
            <td>{{ record.callId }}</td>
            <td>{{ formatDate(record.callTimestamp) }}</td>
            <td>{{ record.csName }}</td>
            <td>{{ record.customerName }}</td>
            <td>
              <span class="badge" :class="getSentimentBadgeClass(record.sentimentScore)">
                {{ record.sentimentScore }}%
              </span>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <div class="pagination" v-if="records.length > 0">
      <div class="pagination-info">
        Showing {{ (paging.page - 1) * paging.size + 1 }} to {{ Math.min(paging.page * paging.size, paging.totalElements) }} of {{ paging.totalElements }} records
      </div>
      <div class="pagination-controls">
        <select v-model="paging.size" class="input-field" style="min-width: 80px; padding: 0.35rem 0.5rem; height: 100%; align-self: center;">
          <option :value="5">5 / page</option>
          <option :value="10">10 / page</option>
          <option :value="20">20 / page</option>
          <option :value="50">50 / page</option>
        </select>
        <button 
          class="btn" 
          :disabled="!paging.hasPrevious" 
          @click="changePage(paging.page - 1)"
        >
          Previous
        </button>
        <button 
          class="btn" 
          :disabled="!paging.hasNext" 
          @click="changePage(paging.page + 1)"
        >
          Next
        </button>
      </div>
    </div>
  </div>
</template>
