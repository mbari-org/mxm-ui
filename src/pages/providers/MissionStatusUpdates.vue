<script setup lang="ts">
import { computed } from 'vue'
import ElapsedTime from 'components/utl/ElapsedTime.vue'

const debug = window.location.search.match(/.*debug=(-|.*\bmissionstatus\b).*/)

interface Props {
  missionStatusUpdates: object[]
}

defineProps<Props>()

const tableConf = computed(() => {
  const columns = [
    {
      field: 'status',
      name: 'status',
      label: 'Status',
      align: 'left',
    },
    {
      field: 'updateDate',
      name: 'updateDate',
      label: 'When',
      align: 'left',
      sortable: true,
    },
  ]

  return {
    columns,
    rowsPerPage: [0],
    pagination: {
      sortBy: 'updateDate',
      descending: true,
      rowsPerPage: 0,
    },
  }
})
</script>

<template>
  <div>
    <q-table
      dense
      :rows="missionStatusUpdates"
      :columns="tableConf.columns"
      row-key="index"
      :rows-per-page-options="tableConf.rowsPerPage"
      :pagination="tableConf.pagination"
      hide-bottom
    >
      <template v-slot:body="props">
        <q-tr>
          <q-td key="status" :props="props">
            {{ props.row.status }}
          </q-td>
          <q-td key="updateDate" :props="props">
            <div class="row q-gutter-x-sm">
              <div>{{ props.row.updateDate }}</div>
              <ElapsedTime
                :date="props.row.updateDate"
                class="text-grey"
                style="font-size: smaller"
              />
            </div>
          </q-td>
        </q-tr>
      </template>
    </q-table>
    <pre v-if="debug">{{ missionStatusUpdates }}</pre>
  </div>
</template>
