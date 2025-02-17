<template>
  <div>
    <div class="wrapper">
      <h2>Ergebnis</h2>
      <div v-if="addressStore.loading">Loading...</div>
      <div v-if="addressStore.error">Error: {{ addressStore.error }}</div>

      <div v-if="!addressStore.loading && addressStore.addresses.length">
        <div class="filter">
          <!-- Search input -->

          <div class="search-input">
            <input
              v-model="searchQuery"
              type="text"
              class="elm-input"
              placeholder="Suche"
              name="search"
              id="search"
              aria-labelledby="search-label"
              data-variant="outline"
            />
            <label class="elm-label" for="search" aria-hidden="true" id="search-label">Suche</label>
          </div>

          <!-- Filter -->
          <div class="select-classification">
            <select
              class="elm-select"
              name="isophone-classification"
              id="isophone-classification"
              data-variant="outline"
              v-model="selectedIsophoneFilterIndex"
            >
              <option value="all">Alle</option>
              <option
                v-for="(label, index) of isophoneLabels.keys()"
                :value="isophoneLabels.get(label)"
                :key="index"
              >
                {{ label }}
              </option>
            </select>
            <label class="elm-label" for="isophone-classification">Einstufung</label>
          </div>
        </div>
      </div>
    </div>

    <!-- Address table -->
    <table>
      <caption class="sr-only">
        Betroffene Adressen im gewählten Bereich. Sortierung:
        {{
          sortDirection === 'desc' ? 'absteigend' : 'aufsteigend'
        }}
        nach
        {{
          sortKey
        }}.
      </caption>
      <thead>
        <tr>
          <th
            @click="sortBy('isophoneLevelDay')"
            :class="getSortClass('isophoneLevelDay')"
            class="center"
            data-icon-before="day"
            data-icon-variant-before="24-outline"
          ></th>
          <th
            @click="sortBy('isophoneLevelNight')"
            :class="getSortClass('isophoneLevelNight')"
            class="center"
            data-icon-before="night"
            data-icon-variant-before="24-outline"
          ></th>
          <th @click="sortBy('city')" :class="getSortClass('city')">Ort</th>
          <th @click="sortBy('postcode')" :class="getSortClass('postcode')">PLZ</th>
          <th @click="sortBy('street')" :class="getSortClass('street')">Straße</th>
          <th class="unsortable small">Nr.</th>
          <th class="unsortable small numeric">Etagen</th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="address in filteredAndSortedAddresses"
          :key="address.id + address.lat + address.lon"
        >
          <td
            :style="{
              background:
                address.isophoneLevelDay === undefined
                  ? 'transparent'
                  : getIsophoneColor(address.isophoneLevelDay, 'day')
            }"
            class="isophone"
          >
            <span v-if="address.isophoneLevelDay !== undefined"
              >> {{ address.isophoneLevelDay }}</span
            >
          </td>
          <td
            :style="{
              background:
                address.isophoneLevelNight === undefined
                  ? 'transparent'
                  : getIsophoneColor(address.isophoneLevelNight, 'night')
            }"
            class="isophone"
          >
            <span v-if="address.isophoneLevelNight !== undefined"
              >> {{ address.isophoneLevelNight }}</span
            >
          </td>
          <td>{{ address.city }}</td>
          <td>{{ address.postcode }}</td>
          <td>{{ address.street }}</td>
          <td class="small">{{ address.housenumber }}</td>
          <td class="small numeric">{{ address.levels }}</td>
        </tr>
      </tbody>
    </table>

    <div class="wrapper">
      <p v-if="!addressStore.loading && filteredAndSortedAddresses.length === 0">
        Keine Adressen gefunden.
      </p>

      <button
        v-if="!addressStore.loading"
        class="elm-button inform"
        data-variant="brand-primary"
        data-width="full"
        title="Betroffene Anwohnende informieren"
        type="button"
        @click="informPeople"
      >
        Betroffene Anwohnende informieren
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useAddressStore } from '../stores/addressStore'
import { computed, onMounted, ref, watch } from 'vue'
import { useConstructionSiteStore } from '../stores/constructionSiteStore'
import { storeToRefs } from 'pinia'
import { getIsophoneColor, isophoneLabels } from '../services/Isophones'

type SortKey = 'postcode' | 'street' | 'city' | 'isophoneLevelDay' | 'isophoneLevelNight'
type SortDirection = 'asc' | 'desc'

const addressStore = useAddressStore()
const constructionSiteStore = useConstructionSiteStore()

const { center, isophonesCalculated, isophonesDay, isophonesNight } =
  storeToRefs(constructionSiteStore)

const lon = computed(() => center.value && center.value[0])
const lat = computed(() => center.value && center.value[1])

const selectedIsophoneFilterIndex = ref<{ day: number; night: number } | 'all'>('all')

// Fetch addresses when the component is mounted
onMounted(() => {
  // addressStore.fetchAddresses(lat.value, lon.value, props.radius)
})

// Watch for changes in the input props and refetch data if they change
watch(
  isophonesCalculated,
  (newValue) => {
    if (newValue && lon.value && lat.value) {
      addressStore.fetchAddresses(lat.value, lon.value, {
        isophonesDay: isophonesDay.value,
        isophonesNight: isophonesNight.value
      })
    }
  },
  { immediate: true }
)

// Search query for filtering addresses
const searchQuery = ref('')

// State for sorting
const sortKey = ref<SortKey>('postcode')
const sortDirection = ref<SortDirection>('asc')

function getSortClass(key: SortKey) {
  return sortKey.value === key ? sortDirection.value : undefined
}

// Method to sort the table by a given column
// Method to sort the table by a given column
function sortBy(key: SortKey) {
  if (sortKey.value === key) {
    sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc'
  } else {
    sortKey.value = key
    sortDirection.value = 'asc'
  }
}

// Filter and sort addresses based on the search query and selected sort column
const filteredAndSortedAddresses = computed(() => {
  const filteredAndSortedResult = [...addressStore.addresses]
    // filter by category
    .filter((address) => {
      if (selectedIsophoneFilterIndex.value === 'all') {
        return true
      }

      return (
        address.isophoneLevelDay === selectedIsophoneFilterIndex.value.day ||
        address.isophoneLevelNight === selectedIsophoneFilterIndex.value.night
      )
    })
  // filter by test input
  /*.filter((address) => {
    const searchLower = searchQuery.value.toLowerCase()
    return (
      [address.street, address.housenumber, address.postcode, address.city]
        .join(' ')
        .toLowerCase()
        .includes(searchLower) ||
      address.postcode?.toString().includes(searchLower) ||
      address.street?.toLowerCase().includes(searchLower) ||
      address.city?.toLowerCase().includes(searchLower)
    )
  })
  .sort((a, b) => {
    const aValue = a[sortKey.value] || ''
    const bValue = b[sortKey.value] || ''

    if (aValue < bValue) return sortDirection.value === 'asc' ? -1 : 1
    if (aValue > bValue) return sortDirection.value === 'asc' ? 1 : -1
    return 0
  })
)*/

  console.log(filteredAndSortedResult)
  return filteredAndSortedResult
})

function informPeople() {
  console.log(filteredAndSortedAddresses.value)
}
</script>

<style scoped>
.search-input {
  flex-grow: 1;

  .elm-input {
    width: 100%;
  }
}

h2 {
  margin: 0.3rem 0.6rem;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead tr {
  vertical-align: bottom;
}

th {
  padding: 0.5rem;
  text-align: left;
  background-color: #f4f4f4;
  border-bottom: 2px solid #ddd;
  position: relative;
}

th:not(.unsortable) {
  cursor: pointer;
}

td {
  padding: 0.5rem;
  border-bottom: 1px solid #ddd;
}

th:hover {
  background-color: #eaeaea;
}

th:after {
  right: 0;
  position: absolute;
  margin-right: 0.5rem;
}

th.asc:after {
  content: '↑';
}

th.desc:after {
  content: '↓';
}

td.isophone {
  text-wrap: nowrap;
  font-size: 0.75rem;
  color: white;
}

th.small,
td.small {
  max-width: 70px;
}

.numeric {
  text-align: right;
}

.center {
  text-align: center;
}

.filter {
  display: flex;
  gap: 0.5rem;
  margin: 1rem 0.2rem;
}

.inform {
  margin-top: 1rem;
}

.wrapper {
  padding: 0 1rem;
}

.select-classification {
  position: relative;
}
</style>
