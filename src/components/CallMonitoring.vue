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
const buildUrl = () => {
  const url = new URL(import.meta.env.VITE_API_URL || '/api/v1/call-monitoring', window.location.origin);
  
  if (search.value) url.searchParams.append('search', search.value);
  url.searchParams.append('page', paging.value.page.toString());
  url.searchParams.append('size', paging.value.size.toString());

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

watch([search, () => paging.value.size], () => {
  paging.value.page = 1; 
  fetchData();
});

watch(() => paging.value.page, () => {
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


    </div>

    <div v-if="error" class="empty-state" style="color: var(--danger)">
      <p>Error: {{ error }}</p>
    </div>

    <div class="table-container">
      <table>
        <thead>
          <tr>
            <th>No.</th>
            <th>Call ID</th>
            <th>Call Timestamp</th>
            <th>CS Name</th>
            <th>Nama Nasabah</th>
            <th>Sentiment Score</th>
          </tr>
        </thead>
        
        <tbody v-if="loading && records.length === 0">
          <tr>
            <td colspan="6" class="empty-state">Loading data...</td>
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


  </div>
</template>
