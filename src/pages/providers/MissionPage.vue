<script setup lang="ts">
import { useRoute } from 'vue-router'
import { useUtlStore } from 'stores/utl'
import { useMutation, useQuery } from '@vue/apollo-composable'

import {
  ARGUMENT_DELETE,
  ARGUMENT_INSERT,
  ARGUMENT_UPDATE,
  MISSION,
  MISSION_DELETE,
  MISSION_UPDATE,
  UNITS,
  UPDATE_MISSION_TEMPLATE,
} from './queries'

import { computed, inject, nextTick, ref, watch } from 'vue'
import ElapsedTime from 'components/utl/ElapsedTime.vue'
import MxmMarkdown from 'components/utl/markdown/MxmMarkdown.vue'
import MissionScheduling from './MissionScheduling.vue'
import UnitSelect from 'pages/providers/UnitSelect.vue'
import { useQuasar } from 'quasar'
import ParameterValue from 'components/paramInput/ParameterValue.vue'

const debug = window.location.search.match(/.*debug=(-|.*\bmission\b).*/)

const { mutate: missionUpdate } = useMutation(MISSION_UPDATE)
const { mutate: missionDelete } = useMutation(MISSION_DELETE)

// used to load parameters when not full template has been loaded yet
const { mutate: missionTemplateUpdate, loading: updatingMissionTemplate } =
  useMutation(UPDATE_MISSION_TEMPLATE)

const { mutate: argumentInsert } = useMutation(ARGUMENT_INSERT)
const { mutate: argumentUpdate } = useMutation(ARGUMENT_UPDATE)
const { mutate: argumentDelete } = useMutation(ARGUMENT_DELETE)

const utl = inject('utl')

const $q = useQuasar()

const route = useRoute()

const params = computed(() => route.params)
const providerId = computed(() => params.value.providerId)
const missionTplId = computed(() => params.value.missionTplId)
const missionId = computed(() => params.value.missionId)

const {
  result: missionResult,
  loading,
  refetch: refetchMission,
} = useQuery(MISSION, { providerId, missionTplId, missionId })

const mission = computed(() => missionResult.value?.mission ?? {})
const provider = computed(() => mission.value?.provider ?? {})
const editable = computed(() => mission.value?.missionStatus === 'DRAFT')

const { result: unitsResult } = useQuery(UNITS, { providerId })

const units = computed(() => unitsResult.value?.unitsForProvider ?? [])

const unitsByName = computed(() =>
  Object.fromEntries(
    units.value.map(u => [
      u.unitName,
      {
        unitName: u.unitName,
        abbreviation: u.abbreviation,
        baseUnit: u.baseUnit,
      },
    ])
  )
)

async function doMissionTemplateUpdate() {
  if (debug) console.debug('doMissionTemplateUpdate')
  $q.loading.show({
    message: 'Reloading template',
    messageColor: 'black',
    customClass: 'text-bold',
  })
  const variables = {
    pl: {
      providerId: providerId.value,
      missionTplId: missionTplId.value,
    },
  }
  try {
    const data = await missionTemplateUpdate(variables)
    if (debug) console.debug('updateMissionTemplate: mutation data=', data)
    $q.notify({
      message: 'Mission template reloaded from provider',
      timeout: 1000,
      position: 'top',
      color: 'info',
    })
    refetchMission()
  } catch (error) {
    console.error('updateMissionTemplate: mutation error=', error)
  } finally {
    $q.loading.hide()
  }
}

watch(mission, async mission => {
  if (!updatingMissionTemplate && !mission?.missionTemplate?.retrievedAt) {
    await doMissionTemplateUpdate()
  }
})

useUtlStore().setRefreshFunction(
  doMissionTemplateUpdate,
  'Refresh this mission'
)

const alreadySavedArgs = computed(() => mission.value?.arguments ?? [])

const parameters = computed(
  () => mission.value?.missionTemplate?.parameters ?? []
)

// TODO
const parametersWithError = computed(() => ({}))
const parametersWithErrorCount = computed(
  () => Object.keys(parametersWithError.value).length
)

const savingArgs = ref(false)
const parametersChanged = computed(() =>
  myArguments.value.filter(
    a => a.paramValue !== a.defaultValue || a.paramUnits !== a.defaultUnits
  )
)

const myArguments = computed(() =>
  parameters.value.map(p => {
    const arg = alreadySavedArgs.value.find(p2 => p2.paramName === p.paramName)
    const paramValue =
      arg && arg.paramValue !== null ? arg.paramValue : p.defaultValue
    const paramUnits = (arg && arg.paramUnits) || p.defaultUnits
    // console.debug('FIND p.paramName=', p.paramName, 'arg=', arg, 'paramValue=', paramValue)
    return {
      paramName: p.paramName,
      type: p.type,
      valueCanReference: p.valueCanReference,
      paramValue,
      paramUnits,
      defaultValue: p.defaultValue,
      defaultUnits: p.defaultUnits,
      required: p.required,
      description: p.description,
    }
  })
)

const disableRun = computed(
  () =>
    mission.value.missionStatus !== 'DRAFT' ||
    parametersWithErrorCount.value > 0 ||
    (provider.value.usesSched &&
      mission.value.schedType === 'DATE' &&
      !mission.value.schedDate)
)

const overriddenFirst = ref(false)

// A workaround to properly reflect the reordering of arguments given that the
// simple toggling of overriddenFirst doesn't seem to propagate all the effects
// (even though that orderedArguments depends on it)
const workaroundForArgsRefresh = ref(true)

function toggleOverriddenFirst() {
  overriddenFirst.value = !overriddenFirst.value
  workaroundForArgsRefresh.value = false
  nextTick(() => {
    workaroundForArgsRefresh.value = true
  })
}

const orderedArguments = computed(() => {
  const overriddenParam = a =>
    a.paramValue !== a.defaultValue || a.paramUnits !== a.defaultUnits

  if (overriddenFirst.value) {
    const ordered = myArguments.value.filter(overriddenParam)
    const others = myArguments.value.filter(a => !overriddenParam(a))
    return ordered.concat(others)
  }
  return myArguments.value
})

function submitMission() {
  console.warn('submitMission() not implemented')
}

function cancelMission() {
  console.warn('cancelMission() not implemented')
}

async function updateMissionSched({ schedType, schedDate }) {
  if (debug)
    console.debug(
      'updateMissionSched: schedType=',
      schedType,
      'schedDate=',
      schedDate
    )
  if (schedType !== 'DATE') {
    schedDate = null
  }

  await updateMission({ schedType, schedDate })
  $q.notify({
    message: 'Mission scheduling updated',
    timeout: 1200,
    color: 'info',
    position: 'top',
  })
}

async function updateMission(missionPatch) {
  if (debug) console.debug('updateMission missionPatch=', missionPatch)
  const variables = {
    pl: {
      ...missionPatch,
      missionId: missionId.value,
      providerId: providerId.value,
      missionTplId: missionTplId.value,
    },
  }
  try {
    const data = await missionUpdate(variables)
    if (debug) console.debug('updateMission: mutation data=', data)
    // TODO apollo mechanism to reflect change, not having to do the refetch necessarily
    refetchMission()
  } catch (error) {
    console.error('updateMission: mutation error=', error)
  }
}

async function updateDescription(description: string) {
  await updateMission({ description })
  $q.notify({
    message: 'Mission description saved',
    timeout: 700,
    color: 'info',
    position: 'top',
  })
}

async function deleteMission() {
  $q.dialog({
    title: 'Confirm',
    message: `Are you sure you want to delete this mission '${missionId.value}'`,
    color: 'negative',
    ok: `Yes, delete '${missionId.value}'`,
    cancel: true,
    focus: 'cancel',
  }).onOk(doDeleteMission)

  async function doDeleteMission() {
    const variables = {
      pl: {
        providerId: providerId.value,
        missionTplId: missionTplId.value,
        missionId: missionId.value,
      },
    }
    try {
      const data = await missionDelete(variables)
      if (debug) console.debug('deleteMission: mutation data=', data)
      $q.notify({
        message: `Mission deleted: '${missionId.value}'`,
        timeout: 2000,
        position: 'top',
        color: 'info',
      })
      utl.replace([providerId.value])
    } catch (error) {
      console.error('deleteMission: mutation error=', error)
      $q.notify({
        message: `Mission deletion error: ${JSON.stringify(error)}`,
        timeout: 0,
        closeBtn: 'Close',
        color: 'warning',
      })
    }
  }
}

async function insertArgument(paramName, paramValue, paramUnits, next) {
  const variables = {
    pl: {
      missionId: missionId.value,
      providerId: providerId.value,
      missionTplId: missionTplId.value,
      paramName,
      paramValue,
      paramUnits,
    },
  }
  if (debug) console.debug('insertArgument: variables=', variables)

  try {
    const data = await argumentInsert(variables)
    if (debug) console.debug('insertArgument: mutation data=', data)
    next(true)
  } catch (error) {
    console.error('insertArgument: mutation error=', error)
    next(false)
  }
}

async function updateArgument(paramName, paramValue, paramUnits, next) {
  if (debug)
    console.debug(
      `updateArgument: paramName=${paramName} paramValue=${paramValue} paramUnits=${paramUnits}`
    )
  const variables = {
    pl: {
      missionId: missionId.value,
      providerId: providerId.value,
      missionTplId: missionTplId.value,
      paramName,
      paramValue,
      paramUnits,
    },
  }
  try {
    const data = await argumentUpdate(variables)
    if (debug) console.debug('updateArgument: mutation data=', data)
    next(true)
  } catch (error) {
    console.error('updateArgument: mutation error=', error)
    next(false)
  }
}

async function deleteArgument(paramName, next) {
  const variables = {
    pl: {
      missionId: missionId.value,
      providerId: providerId.value,
      missionTplId: missionTplId.value,
      paramName,
    },
  }
  try {
    const data = await argumentDelete(variables)
    if (debug) console.debug('deleteArgument: mutation data=', data)
    next(true)
  } catch (error) {
    console.error('deleteArgument: mutation error=', error)
    next(false)
  }
}

function saveArguments(row) {
  if (debug) console.debug('saveArguments: row=', row)
  if (savingArgs.value) {
    return
  }

  savingArgs.value = true

  let numInserted = 0
  let numUpdated = 0
  let numDeleted = 0

  const argList = [...myArguments.value]
  const nextArg = () => {
    const arg = argList.pop()
    if (!arg) {
      if (debug)
        console.debug(
          `saveArguments DONE: numInserted=${numInserted} numUpdated=${numUpdated} numDeleted=${numDeleted}`
        )

      savingArgs.value = false
      if (numInserted || numUpdated || numDeleted) {
        refetchMission()
        $q.notify({
          message: 'Arguments updated',
          timeout: 700,
          position: 'top',
          color: 'info',
        })
      }
      return
    }

    if (debug)
      console.debug(
        `saveArguments: checking ${arg.paramName}`,
        `v='${arg.paramValue}' dv='${arg.defaultValue}'`
      )

    const alreadySavedArg = alreadySavedArgs.value.find(
      x => x.paramName === arg.paramName
    )

    if (debug) console.debug(arg.paramName, 'alreadySavedArg=', alreadySavedArg)

    if (
      arg.paramValue !== arg.defaultValue ||
      arg.paramUnits !== arg.defaultUnits
    ) {
      if (alreadySavedArg) {
        if (
          alreadySavedArg.paramValue !== arg.paramValue ||
          alreadySavedArg.paramUnits !== arg.paramUnits
        ) {
          if (debug) console.debug(arg.paramName, 'UPDATING', arg.paramValue)
          updateArgument(
            alreadySavedArg.paramName,
            arg.paramValue,
            arg.paramUnits,
            ok => {
              if (ok) {
                numUpdated++
              }
              nextArg()
            }
          )
        } else nextArg()
      } else {
        if (debug) console.debug(arg.paramName, 'INSERTING', arg.paramValue)
        insertArgument(arg.paramName, arg.paramValue, arg.paramUnits, ok => {
          if (ok) {
            numInserted++
          }
          nextArg()
        })
      }
    } else {
      // arg has the default value.
      if (alreadySavedArg) {
        if (debug) console.debug(arg.paramName, 'DELETING', arg.paramName)
        deleteArgument(alreadySavedArg.paramName, ok => {
          if (ok) {
            numDeleted++
          }
          nextArg()
        })
      } else nextArg()
    }
  }

  if (debug) console.debug('saveArguments START')
  nextArg()
}

const filter = ref('')

const tableConf = computed(() => {
  const columns = [
    {
      field: 'paramName',
      name: 'paramName',
      label: 'Parameter',
      align: 'left',
    },
    {
      field: 'paramValue',
      name: 'paramValue',
      label: 'Value',
      align: 'left',
    },
  ]
  if (mission.value?.provider?.usesUnits) {
    columns.push({
      field: 'paramUnits',
      name: 'paramUnits',
      label: 'Units',
      align: 'left',
    })
  }
  columns.push({
    field: 'description',
    name: 'description',
    label: 'Description',
    align: 'left',
  })

  return {
    columns,
    rowsPerPage: [0],
    pagination: {
      rowsPerPage: 0,
    },
  }
})
</script>

<template>
  <q-page class="q-pa-md">
    <div v-if="!loading && mission.missionId">
      <q-card class="q-mb-md">
        <q-card-section>
          <div class="row no-wrap q-gutter-x-lg">
            <div class="column q-gutter-y-sm">
              <div class="row no-wrap items-center q-gutter-x-sm">
                <div>Mission:</div>
                <div class="text-bold">{{ mission.missionId }}</div>
                <div class="row no-wrap items-center text-grey q-gutter-x-sm">
                  <div>Status:</div>
                  <div class="text-bold">{{ mission.missionStatus }}</div>
                  <div style="font-size: smaller">
                    <ElapsedTime
                      :date="mission.updatedDate"
                      tooltip="Time of last status update"
                    />
                  </div>
                </div>
              </div>
              <div
                class="row no-wrap items-center q-gutter-x-sm"
                style="font-size: smaller"
              >
                <div class="text-gray">Template:</div>
                <div>
                  <router-link
                    class="appLink"
                    :to="
                      utl.routeLoc([
                        mission.providerId,
                        'mt',
                        mission.missionTplId,
                      ])
                    "
                  >
                    {{ mission.missionTplId }}
                  </router-link>
                </div>

                <div class="text-gray">Asset:</div>
                <div>
                  <router-link
                    class="appLink"
                    :to="
                      utl.routeLoc([params.providerId, 'a', mission.assetId])
                    "
                  >
                    {{ mission.assetId }}
                    <q-tooltip>
                      {{ mission.asset.className }}
                    </q-tooltip>
                  </router-link>
                </div>
              </div>
              <div class="row text-grey" style="font-size: smaller">
                <ElapsedTime
                  :date="mission?.missionTemplate?.retrievedAt"
                  tooltip="Time of last template update"
                />
              </div>
              <div
                class="row no-wrap items-center q-gutter-x-sm"
                style="font-size: smaller"
              >
                <div>PMID:</div>
                <div>
                  {{ mission.providerMissionId || '??' }}
                </div>
                <q-tooltip>Provider mission ID, if already assigned</q-tooltip>
              </div>
            </div>
            <MxmMarkdown
              expandable
              expandable-title="Description:"
              :text="mission.description"
              :start-markdown="provider.descriptionFormat === 'markdown'"
              :editable="editable"
              @saveDescription="updateDescription"
            />
          </div>
          <table class="mission-table" style="font-size: smaller">
            <tbody>
              <tr v-if="mission.start">
                <td>Start:</td>
                <td>{{ mission.start }}</td>
              </tr>
              <tr v-if="mission.end">
                <td>End:</td>
                <td>{{ mission.end }}</td>
              </tr>
            </tbody>
          </table>
        </q-card-section>
      </q-card>

      <div
        v-if="provider && provider.usesSched"
        class="row q-mb-sm q-gutter-x-lg"
      >
        <MissionScheduling
          :sched-info="{
            schedType: mission.schedType,
            schedDate: mission.schedDate,
          }"
          :editable="mission.missionStatus === 'DRAFT'"
          @schedInfo="updateMissionSched"
        />
      </div>

      <div class="row q-mb-sm q-gutter-x-lg">
        <q-btn
          v-if="provider.canValidate"
          label="Validate"
          icon="check"
          push
          color="secondary"
          size="sm"
          :disable="
            mission.missionStatus !== 'DRAFT' || parametersWithErrorCount > 0
          "
          @click="validateMission"
        >
          <q-tooltip>Validate mission against external provider</q-tooltip>
        </q-btn>
        <q-btn
          label="Submit"
          icon="settings"
          push
          color="secondary"
          size="sm"
          :disable="disableRun"
          @click="submitMission"
        >
          <q-tooltip>Request execution of this mission</q-tooltip>
        </q-btn>
        <q-btn
          v-if="
            mission.missionStatus === 'RUNNING' ||
            mission.missionStatus === 'QUEUED'
          "
          label="Cancel"
          icon="cancel"
          push
          color="secondary"
          size="sm"
          @click="cancelMission"
        >
          <q-tooltip>Request cancellation of submitted mission</q-tooltip>
        </q-btn>
        <q-btn
          label="Delete"
          icon="delete"
          push
          color="secondary"
          size="sm"
          :disable="
            mission.missionStatus !== 'DRAFT' &&
            mission.missionStatus !== 'TERMINATED'
          "
          @click="deleteMission"
        >
          <q-tooltip>
            Delete this mission (only allowed if in DRAFT or TERMINATED status)
          </q-tooltip>
        </q-btn>
      </div>

      <q-table
        v-if="workaroundForArgsRefresh"
        :dense="$q.screen.lt.md"
        :rows="orderedArguments"
        :columns="tableConf.columns"
        row-key="name"
        :rows-per-page-options="tableConf.rowsPerPage"
        :pagination="tableConf.pagination"
        :filter="filter"
        separator="cell"
        :no-data-label="`No parameters defined in the mission template '${mission.missionTplId}'`"
      >
        <template v-slot:top>
          <div class="col full-width">
            <div class="row q-gutter-lg no-wrap">
              <div class="row vertical-middle text-weight-light text-grey">
                <div :class="{ 'text-green': parametersChanged.length }">
                  Overridden parameters: {{ parametersChanged.length }}
                </div>
                <q-btn
                  v-if="parametersChanged.length"
                  class="q-ml-md"
                  icon="arrow_upward"
                  dense
                  round
                  color="green-4"
                  size="xs"
                  :outline="!overriddenFirst"
                  @click="toggleOverriddenFirst"
                >
                  <q-tooltip>Show overridden parameters first</q-tooltip>
                </q-btn>
              </div>
              <div
                v-if="parametersWithErrorCount"
                class="vertical-middle text-red"
              >
                Arguments missing or with error: {{ parametersWithErrorCount }}
              </div>
            </div>

            <div class="full-width row items-center">
              <div class="col-auto text-h5">Arguments</div>

              <div class="q-ml-md row">
                <q-input
                  v-if="myArguments.length"
                  class="col"
                  color="secondary"
                  v-model="filter"
                  placeholder="Filter"
                  clearable
                />
              </div>
            </div>
          </div>
        </template>

        <template v-slot:body="props">
          <q-tr :props="props">
            <q-td
              key="paramName"
              :props="props"
              style="width: 5px; font-family: monospace; vertical-align: top"
              class="cursor-pointer"
              @dblclick="
                $router.push(
                  utl.routeLoc([
                    providerId,
                    'mt',
                    missionTplId,
                    'p',
                    props.row.paramName,
                  ])
                )
              "
            >
              <q-tooltip v-if="true">
                <pre>row={{ props.row }}</pre>
                <pre v-if="unitsByName.length">
unit={{ unitsByName[props.row.paramUnits] }}</pre
                >
              </q-tooltip>
              <div
                style="font-size: 1.1em"
                :class="`${props.row.required ? 'text-bold' : ''}`"
              >
                {{ props.row.paramName }}
              </div>

              <div class="text-grey-7 q-mt-sm" style="font-size: 0.8em">
                {{ props.row.type }}
                <span v-if="props.row.valueCanReference">
                  | {{ props.row.valueCanReference }}
                </span>
              </div>
            </q-td>

            <q-td
              key="paramValue"
              :props="props"
              style="width: 12em; font-family: monospace; vertical-align: top"
              class="paramValueCell"
            >
              <ParameterValue
                class="q-pa-xs"
                style="
                  font-family: monospace;
                  width: 12em;
                  word-break: break-all;
                  font-size: 0.9em;
                  overflow-x: auto;
                "
                :paramRequired="props.row.required"
                :label="`${props.row.paramName}:`"
                :paramName="props.row.paramName"
                :paramType="props.row.type"
                :valueCanReference="props.row.valueCanReference"
                :paramValue="props.row.paramValue"
                :defaultValue="props.row.defaultValue"
                :editable="editable"
                @valueInput="
                  val => {
                    delete parametersWithError[props.row.paramName]
                    props.row.paramValue = val
                    saveArguments(props.row)
                  }
                "
                @valueError="
                  error => {
                    parametersWithError[props.row.paramName] = error
                  }
                "
              />
            </q-td>

            <q-td
              v-if="provider && provider.usesUnits"
              key="paramUnits"
              :props="props"
              style="vertical-align: top; width: 10em"
              class="paramValueCell"
            >
              <UnitSelect
                :units="units"
                :unitsByName="unitsByName"
                :value="props.row.paramUnits"
                :resetValue="props.row.defaultUnits"
                :editable="editable"
                @input="
                  val => {
                    props.row.paramUnits = val
                    saveArguments(props.row)
                  }
                "
              >
                <q-tooltip
                  v-if="debug && props.row.paramUnits"
                  anchor="bottom left"
                  self="bottom right"
                >
                  <pre>{{ unitsByName[props.row.paramUnits] }}</pre>
                </q-tooltip>
              </UnitSelect>
            </q-td>

            <q-td key="description" :props="props" style="vertical-align: top">
              <MxmMarkdown
                simple
                hide-empty
                :text="props.row.description"
                :start-markdown="provider.descriptionFormat === 'markdown'"
              />
            </q-td>
          </q-tr>
        </template>
      </q-table>
    </div>
    <i v-else-if="loading"> Loading... </i>
    <div v-else>
      <div class="text-negative">Not found:</div>
      <table class="q-ml-md">
        <tbody>
          <tr>
            <td>Mission:</td>
            <td />
            <td>{{ params.missionId }}</td>
          </tr>
          <tr>
            <td>Mission Template:</td>
            <td />
            <td>{{ params.missionTplId }}</td>
          </tr>
          <tr>
            <td>Provider:</td>
            <td />
            <td>{{ params.providerId }}</td>
          </tr>
        </tbody>
      </table>
    </div>

    <div v-if="debug">
      <pre>loading={{ loading }}</pre>
      <pre>MissionPage: mission={{ mission }}</pre>
    </div>
  </q-page>
</template>

<style scoped>
.mission-table td {
  padding: 2px 4px;
  vertical-align: top;
}

body.body--dark .paramValueCell:hover {
  background-color: #333333;
}

.paramValueCell:hover {
  background-color: #eeeeee;
}
</style>