<template>
  <div class="h-screen flex flex-col justify-center items-center">
    <div>Dataset: {{ selectedDataset }}</div>
    <div>Metric: {{ selectedMetric }}</div>
  </div>
</template>

<script lang="ts" setup>
import { ref, unref } from 'vue'
import '@leanix/reporting'

const getDatasets = (): Array<lxr.DropdownEntry & { config: string }> => ([
  {id:'1', label:'Azure resources Feb 16 Prod', config:'az_res-20240216.json'},
  {id:'5', label:'Azure resources Feb 21 Prod', config:'az_res-20240221.json'},
  {id:'2', label:'Azure resources Feb 08 Prod', config:'az_res-20240208.json'},
  {id:'3', label:'Azure resources Jan 26 Sandbox', config:'az_res_with_parents_sb-20240126.json'},
  {id:'4', label:'Azure resources Dec 19', config:'azure_resources_by_mg-20231219.json'}
])

const selectedDataset = ref(getDatasets()[0].id)
const selectedMetric = ref('0') // first value for this variable is irrelevant, will be updated in the reportUIUpdate method

const getMetrics = (dataset: string): Array<lxr.DropdownEntry> => {
  return [
    { id: `${dataset}:0`, label: `Metric ${dataset}:0` },
    { id: `${dataset}:1`, label: `Metric ${dataset}:1` }
  ]
}

const getUIElements = (): lxr.UIElements => {
  const metrics = getMetrics(unref(selectedDataset))
  const idxMetric = metrics.findIndex(({ id }) => id === unref(selectedMetric))
  if (idxMetric < 0) selectedMetric.value = metrics[0].id
  return {
    root:{
      items:[
        {
          disabled: false,
          entries: getDatasets(),
          id: 'dataset',
          label: 'Dataset',
          type: 'dropdown'
        },
        {
          disabled: false,
          entries: metrics,
          id: 'metric',
          label: 'Metric',
          type: 'dropdown'
        }
      ]
    },
    values: {
      dataset: unref(selectedDataset),
      metric: unref(selectedMetric)
    }
  }

}

const reportUIUpdate = (selection: lxr.UISelection): lxr.UIMinimalConfiguration | undefined => {
  const { dataset } = selection.elements?.values as { dataset: string, metric: string }
  // if dataset has changed, we'll return an updated lxr.UIMinimalConfiguration with the new metrics dropdown
  if (unref(selectedDataset) !== dataset) {
    selectedDataset.value = dataset
    const elements = getUIElements()
    return { elements }
  }
  return undefined
}

const createConfig = (): lxr.ReportConfiguration => {
  return {
    menuActions: {
      showConfigure: false
    },
    ui: {
      elements: getUIElements(),
      update: reportUIUpdate
    }
  }
}

const fetchImpactGraphQL = async () => {
  const url = new URL(lx.currentSetup.settings.baseUrl)
  url.pathname = 'services/impacts/v1/graphql'
  const query = 'allFactSheets { edges { node { id } } } '
  // Doesn't work, we get a 415 - Unsupported Media Type because lx.executeParentOriginXHR doesn't allow to set the
  // header "Content-Type": "application/json"
  const res = await lx.executeParentOriginXHR('POST', url.toString(), JSON.stringify({ query }))
  console.log('GOT RESPONSE', res)
}

const initReport = async () => {
  await lx.init()
  await lx.ready(createConfig())
  await fetchImpactGraphQL()
}

initReport()
</script>
