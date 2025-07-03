<script setup lang="ts">
import {
  QTable,
  QInput,
  QSelect,
  QBtn,
  QDialog,
  QCard,
  QCardActions,
  QCardSection,
  QTd,
  QDate,
  QPopupProxy,
  QIcon,
} from 'quasar'
import { ref, computed, onMounted, nextTick } from 'vue'
import moment from 'moment'
import { Chart, registerables } from 'chart.js'
import db from '@/db/db.json'

Chart.register(...registerables)

interface Device {
  id: number
  name: string
  type: string
  status: string
  last_active?: string
}

interface Reading {
  id: number
  device_id: number
  timestamp: string
  value: number
}

const allDevices = ref<Device[]>([])
const search = ref('')
const filterType = ref('')
const filterStatus = ref('')

const deviceTypes = Array.from(new Set(db.devices.map((d) => d.type)))
const statuses = ['online', 'offline', 'unknown']

const isModalOpen = ref(false)
const currentDevice = ref<Partial<Device>>({})

function openAdd() {
  currentDevice.value = {
    name: '',
    type: deviceTypes[0] || '',
    status: statuses[0],
    last_active: moment().format('YYYY-MM-DD HH:mm'),
  }
  isModalOpen.value = true
}

function openEdit(device: Device) {
  currentDevice.value = { ...device }
  isModalOpen.value = true
}

function saveDevice() {
  if (!currentDevice.value.name) return

  if (currentDevice.value.id) {
    const idx = allDevices.value.findIndex((d) => d.id === currentDevice.value.id)
    if (idx !== -1) {
      allDevices.value[idx] = {
        ...(currentDevice.value as Device),
        last_active: allDevices.value[idx].last_active,
      }
    }
  } else {
    const newId = Math.max(...allDevices.value.map((d) => d.id), 0) + 1
    allDevices.value.push({
      id: newId,
      name: currentDevice.value.name || '',
      type: currentDevice.value.type || '',
      status: currentDevice.value.status || '',
      last_active: moment().format('YYYY-MM-DD HH:mm'),
    })
  }

  isModalOpen.value = false
}

const filteredDevices = computed(() => {
  return allDevices.value.filter((device) => {
    const nameMatch = device.name.toLowerCase().includes(search.value.toLowerCase())
    const typeMatch = !filterType.value || device.type === filterType.value
    const statusMatch = !filterStatus.value || device.status === filterStatus.value
    return nameMatch && typeMatch && statusMatch
  })
})

onMounted(() => {
  allDevices.value = db.devices.map((d) => ({
    ...d,
    last_active: moment(d.last_active).format('YYYY-MM-DD HH:mm'),
  }))
})

const isChartModalOpen = ref(false)
const selectedDevice = ref<Device | null>(null)

const dateFrom = ref<string | null>(null)
const dateTo = ref<string | null>(null)

const deviceReadings = ref<Reading[]>([])
const filteredReadings = ref<Reading[]>([])

const chartCanvas = ref<HTMLCanvasElement | null>(null)
let chartInstance: Chart | null = null

function openChart(device: Device) {
  selectedDevice.value = device

  const readingsForDevice = db.device_readings.filter((r) => r.device_id === device.id)

  deviceReadings.value = readingsForDevice.map((r) => ({
    ...r,
    timestamp: r.timestamp,
  }))

  if (deviceReadings.value.length > 0) {
    deviceReadings.value.sort(
      (a, b) => new Date(a.timestamp).getTime() - new Date(b.timestamp).getTime(),
    )
    dateFrom.value = moment(deviceReadings.value[0].timestamp).format('YYYY-MM-DD')
    dateTo.value = moment(deviceReadings.value[deviceReadings.value.length - 1].timestamp).format(
      'YYYY-MM-DD',
    )
  } else {
    dateFrom.value = null
    dateTo.value = null
  }

  filterReadings()
  isChartModalOpen.value = true
}

function filterReadings() {
  if (!dateFrom.value || !dateTo.value) {
    filteredReadings.value = deviceReadings.value.slice()
    return
  }

  const fromTs = new Date(dateFrom.value).getTime()
  const toTs = new Date(dateTo.value).getTime() + 24 * 3600 * 1000 - 1 // include full day

  filteredReadings.value = deviceReadings.value.filter((r) => {
    const ts = new Date(r.timestamp).getTime()
    return ts >= fromTs && ts <= toTs
  })

  nextTick(() => {
    renderChart()
  })
}

function renderChart() {
  if (!chartCanvas.value) return

  if (chartInstance) {
    chartInstance.destroy()
  }

  const labels = filteredReadings.value.map((r) => moment(r.timestamp).format('YYYY-MM-DD HH:mm'))
  const data = filteredReadings.value.map((r) => r.value)

  chartInstance = new Chart(chartCanvas.value, {
    type: 'line',
    data: {
      labels,
      datasets: [
        {
          label: 'Стойност',
          data,
          borderColor: 'rgba(54, 162, 235, 0.8)',
          backgroundColor: 'rgba(54, 162, 235, 0.2)',
          fill: true,
          tension: 0.3,
          pointRadius: 3,
          pointHoverRadius: 6,
        },
      ],
    },
    options: {
      responsive: true,
      interaction: {
        mode: 'nearest',
        intersect: false,
      },
      scales: {
        x: {
          title: { display: true, text: 'Време' },
          ticks: { maxRotation: 45, minRotation: 45 },
        },
        y: {
          title: { display: true, text: 'Стойност' },
        },
      },
    },
  })
}
</script>

<template>
  <div class="q-pa-md">
    <div class="flex row justify-between items-center">
      <div class="q-gutter-md row items-center">
        <q-input v-model="search" label="Търсене по име" dense debounce="300" style="width: 100%" />
        <q-select
          v-model="filterType"
          :options="deviceTypes"
          label="Филтър по тип"
          dense
          clearable
          style="width: 100%"
        />
        <q-select
          v-model="filterStatus"
          :options="statuses"
          label="Филтър по статус"
          dense
          clearable
          style="width: 100%"
        />
      </div>
      <q-btn label="Добави устройство" color="primary" @click="openAdd" />
    </div>

    <q-table
      class="q-mt-md"
      :rows="filteredDevices"
      :columns="[
        { name: 'actions', label: '', field: 'actions', align: 'center' },
        { name: 'name', label: 'Име', field: 'name', sortable: true },
        { name: 'type', label: 'Тип', field: 'type', sortable: true },
        { name: 'status', label: 'Статус', field: 'status', sortable: true },
        { name: 'last_active', label: 'Последна активност', field: 'last_active', sortable: true },
      ]"
      row-key="id"
      flat
      bordered
      @row-click="(evt, row) => openChart(row)"
    >
      <template #body-cell-actions="props">
        <q-td>
          <q-btn dense flat icon="edit" color="primary" @click.stop="openEdit(props.row)" />
        </q-td>
      </template>
    </q-table>

    <!-- Добави/Промени устройство -->
    <q-dialog v-model="isModalOpen" persistent>
      <q-card style="min-width: 400px">
        <q-card-section>
          <div class="text-h6">{{ currentDevice.id ? 'Редакция' : 'Добавяне' }}</div>
        </q-card-section>

        <q-card-section class="q-gutter-md">
          <q-input
            v-model="currentDevice.name"
            label="Име"
            :rules="[(val) => !!val || 'Изисква се име']"
            dense
            filled
          />
          <q-select
            v-model="currentDevice.type"
            :options="deviceTypes"
            label="Тип"
            dense
            filled
            emit-value
            map-options
          />
          <q-select
            v-model="currentDevice.status"
            :options="statuses"
            label="Статус"
            dense
            filled
            emit-value
            map-options
          />
        </q-card-section>

        <q-card-actions align="right">
          <q-btn flat label="Отказ" color="primary" @click="isModalOpen = false" />
          <q-btn flat label="Запази" color="primary" @click="saveDevice" />
        </q-card-actions>
      </q-card>
    </q-dialog>

    <!-- Устройство - детайли/графика -->
    <q-dialog v-model="isChartModalOpen" persistent maximized>
      <q-card style="min-width: 600px; max-width: 90vw; max-height: 90vh; overflow-y: auto">
        <q-card-section>
          <div class="text-h6">Устройство: {{ selectedDevice?.name }}</div>
          <div>Тип: {{ selectedDevice?.type }} | Статус: {{ selectedDevice?.status }}</div>
          <div>Последна активност: {{ selectedDevice?.last_active }}</div>
        </q-card-section>

        <q-card-section>
          <div class="row q-gutter-sm items-center">
            <q-input filled v-model="dateFrom" mask="date" :rules="['date']">
              <template v-slot:append>
                <q-icon name="event" class="cursor-pointer">
                  <q-popup-proxy cover transition-show="scale" transition-hide="scale">
                    <q-date
                      v-model="dateFrom"
                      mask="YYYY-MM-DD"
                      label="От"
                      :max="dateTo || undefined"
                      dense
                      style="max-width: 150px"
                      @update:model-value="filterReadings"
                      ><div class="row items-center justify-end">
                        <q-btn v-close-popup label="Close" color="primary" flat /></div
                    ></q-date>
                  </q-popup-proxy>
                </q-icon>
              </template>
            </q-input>
            <q-input filled v-model="dateTo" mask="date" :rules="['date']">
              <template v-slot:append>
                <q-icon name="event" class="cursor-pointer">
                  <q-popup-proxy cover transition-show="scale" transition-hide="scale">
                    <q-date
                      v-model="dateTo"
                      mask="YYYY-MM-DD"
                      label="До"
                      :min="dateFrom || undefined"
                      dense
                      style="max-width: 150px"
                      @update:model-value="filterReadings"
                      ><div class="row items-center justify-end">
                        <q-btn v-close-popup label="Close" color="primary" flat /></div
                    ></q-date>
                  </q-popup-proxy>
                </q-icon>
              </template>
            </q-input>
            <q-btn label="Филтрирай" color="primary" @click="filterReadings" dense />
          </div>
          <canvas ref="chartCanvas" style="max-width: 100%; margin-top: 20px" />
        </q-card-section>

        <q-card-actions align="right">
          <q-btn flat label="Затвори" color="primary" @click="isChartModalOpen = false" />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </div>
</template>
