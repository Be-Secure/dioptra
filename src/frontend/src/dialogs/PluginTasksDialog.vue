<template>
  <q-dialog v-model="showDialog">
    <q-card flat style="min-width: 500px; max-width: 90vw; width: auto;">
      <q-card-section class="bg-primary text-white q-mb-md">
        <div class="text-h6 row justify-between">
          Import Plugin Tasks
        </div>
      </q-card-section>

      <q-form @submit="submit()">
        <q-card-section>
          <p class="text-body2">
            The plugin tasks below have been inferred from your python code. <br>
            Select the tasks you would like to import.
          </p>
          <p v-if="errorMessage" class="text-negative" style="max-width: 500px;">
            Error: {{ errorMessage }}
          </p>
          <TableComponent
            :rows="tasks"
            :columns="taskColumns"
            title="Plugin Tasks"
            ref="tableRef"
            :hideToggleDraft="true"
            :hideCreateBtn="true"
            :hideSearch="true"
            :disableSelect="true"
            :hideOpenBtn="true"
            :hideDeleteBtn="true"
          >
            <template #body-cell-name="props">
            <div style="font-size: 18px;">
              {{ props.row.name }}
              <q-btn icon="edit" round size="sm" color="primary" flat />
              <p v-if="props.row.missing_types.length > 0" class="text-caption text-negative">
                Missing Types:
                <div v-for="type in props.row.missing_types">
                  {{ type.name }}
                </div>
              </p>
            </div>
              <q-popup-edit v-model="props.row.name" v-slot="scope">
                <q-input v-model="scope.value" dense autofocus counter @keyup.enter="scope.set" />
              </q-popup-edit>
            </template>
            <template #body-cell-inputParams="props">
              <div class="column items-end">
                <q-chip
                  v-for="(param, i) in props.row.inputs"
                  :key="i"
                  color="indigo"
                  text-color="white"
                  dense
                >
                  {{ `${param.name}` }}
                  <span v-if="param.required" class="text-red">*</span>
                  {{ `: ${param.type}` }}
                </q-chip>
              </div>
            </template>
            <template #body-cell-outputParams="props">
              <div class="column items-end">
              <q-chip
                v-for="(param, i) in props.row.outputs"
                :key="i"
                color="purple"
                text-color="white"
                dense
                :label="`${param.name}: ${param.type}`"
              />
              </div>
            </template>
            <template #body-cell-select="props">
              <q-checkbox
                v-model="selectedTasks"
                :val="props.row"
              />
            </template>
          </TableComponent>
        </q-card-section>

        <q-separator />

        <q-card-actions align="right" class="text-primary">
          <q-btn 
            outline
            color="primary cancel-btn" 
            label="Cancel" 
            v-close-popup 
            class="q-mr-xs"
          />
           <q-btn
            color="primary"
            type="submit"
            >
              Import
           </q-btn>
        </q-card-actions>
      </q-form>
    </q-card>
  </q-dialog>
</template>

<script setup>
import { inject, ref, watch } from 'vue'
import * as api from '@/services/dataApi'
import TableComponent from '@/components/TableComponent.vue'
import * as notify from '../notify'

const props = defineProps(['pythonCode', 'pluginParameterTypes'])
const emit = defineEmits(['addTasks'])

const isMedium = inject('isMedium')
const isMobile = inject('isMobile')
const isExtraSmall = inject('isExtraSmall')

const showDialog = defineModel()

watch(() => showDialog.value, (newVal) => {
  if(newVal) {
    tasks.value = []
    errorMessage.value = ""
    suggestPluginTasks()
  }
})

const tasks = ref([])
const selectedTasks = ref([])
const errorMessage = ref('')

async function suggestPluginTasks() {
  try {
    const res = await api.suggestPluginTasks(props.pythonCode)
    tasks.value = res.data.tasks
    selectedTasks.value = res.data.tasks
  } catch(err) {
    console.warn(err)
    notify.error(err.response.data.message)
    errorMessage.value = err.response.data.message
  }
}

const taskColumns = [
  { name: 'select', label: 'Select', align: 'center', },
  { name: 'name', label: 'Name', align: 'left', field: 'name', sortable: false, classes: 'vertical-top', },
  { name: 'inputParams', label: 'Input Params', field: 'inputParams', align: 'right', sortable: false, classes: 'vertical-top', },
  { name: 'outputParams', label: 'Output Params', field: 'outputParams', align: 'right', sortable: false, classes: 'vertical-top', },
]

async function submit() {
  let processedTasks = selectedTasks.value.map((task) => {
    const { inputs, outputs, name } = task;
    return {
      name,
      inputParams: inputs,
      outputParams: outputs
    }
  })

  processedTasks.forEach((task) => {
    [...task.inputParams, ...task.outputParams].forEach((param) => {
      param.parameterType = props.pluginParameterTypes.find((type) => type.name === param.type)?.id
      delete param.type
    })
  })
  emit('addTasks', processedTasks)
  showDialog.value = false
}

</script>