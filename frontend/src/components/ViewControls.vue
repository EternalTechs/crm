<template>
  <div class="flex items-center justify-between px-5 pb-4 pt-3">
    <div class="flex items-center gap-2">
      <Dropdown :options="viewsDropdownOptions">
        <template #default="{ open }">
          <Button :label="currentView.label">
            <template #prefix>
              <FeatherIcon :name="currentView.icon" class="h-4" />
            </template>
            <template #suffix>
              <FeatherIcon
                :name="open ? 'chevron-up' : 'chevron-down'"
                class="h-4 text-gray-600"
              />
            </template>
          </Button>
        </template>
      </Dropdown>
    </div>
    <div class="flex items-center gap-2">
      <div v-if="viewUpdated" class="flex items-center gap-2 border-r pr-2">
        <Button label="Cancel" @click="cancelChanges" />
        <Button
          :label="view?.name ? 'Save Changes' : 'Create View'"
          @click="saveView"
        />
      </div>
      <div class="flex items-center gap-2">
        <Filter
          v-model="list"
          :doctype="doctype"
          :default_filters="filters"
          @update="updateFilter"
        />
        <SortBy v-model="list" :doctype="doctype" @update="updateSort" />
        <ViewSettings
          v-model="list"
          :doctype="doctype"
          @update="(isDefault) => updateColumns(isDefault)"
        />
        <Dropdown :options="viewActions">
          <template #default>
            <Button>
              <FeatherIcon name="more-horizontal" class="h-4 w-4" />
            </Button>
          </template>
        </Dropdown>
      </div>
    </div>
  </div>
  <ViewModal
    :doctype="doctype"
    :view="view"
    :options="{
      afterCreate: (v) => {
        viewUpdated = false
        router.push({ name: route.name, query: { view: v.name } })
      },
      afterUpdate: (v) => {
        viewUpdated = false
        currentView = {
          label: v.label,
          icon: v.icon || 'list',
        }
      },
    }"
    v-model="showViewModal"
  />
</template>
<script setup>
import DuplicateIcon from '@/components/Icons/DuplicateIcon.vue'
import PinIcon from '@/components/Icons/PinIcon.vue'
import UnpinIcon from '@/components/Icons/UnpinIcon.vue'
import ViewModal from '@/components/Modals/ViewModal.vue'
import SortBy from '@/components/SortBy.vue'
import Filter from '@/components/Filter.vue'
import ViewSettings from '@/components/ViewSettings.vue'
import { globalStore } from '@/stores/global'
import { viewsStore } from '@/stores/views'
import { useDebounceFn } from '@vueuse/core'
import { createResource, FeatherIcon, Dropdown, call } from 'frappe-ui'
import { computed, ref, defineModel, onMounted, watch, h } from 'vue'
import { useRouter, useRoute } from 'vue-router'

const props = defineProps({
  doctype: {
    type: String,
    required: true,
  },
  filters: {
    type: Object,
    default: {},
  },
})

const { $dialog } = globalStore()
const { reload: reloadView, getView } = viewsStore()

const list = defineModel()

const route = useRoute()
const router = useRouter()

const defaultParams = ref('')

const viewUpdated = ref(false)
const showViewModal = ref(false)

const currentView = computed(() => {
  return {
    label: view.value.label || 'List View',
    icon: view.value.icon || 'list',
  }
})

const view = ref({
  name: '',
  label: '',
  filters: props.filters,
  order_by: 'modified desc',
  columns: '',
  rows: '',
  default_columns: false,
  pinned: false,
})

function getParams() {
  let _view = getView(route.query.view)
  const filters = (_view?.filters && JSON.parse(_view.filters)) || props.filters
  const order_by = _view?.order_by || 'modified desc'
  const columns = _view?.columns || ''
  const rows = _view?.rows || ''

  if (_view) {
    view.value = {
      name: _view.name,
      label: _view.label,
      filters: _view.filters,
      order_by: _view.order_by,
      columns: _view.columns,
      rows: _view.rows,
      route_name: _view.route_name,
      default_columns: _view.row,
      pinned: _view.pinned,
    }
  } else {
    view.value = {
      name: '',
      label: '',
      filters: props.filters,
      order_by: 'modified desc',
      columns: '',
      rows: '',
      route_name: '',
      default_columns: true,
      pinned: false,
    }
  }

  return {
    doctype: props.doctype,
    filters: filters,
    order_by: order_by,
    columns: columns,
    rows: rows,
    custom_view_name: _view?.name || '',
  }
}

list.value = createResource({
  url: 'crm.api.doc.get_list_data',
  params: getParams(),
  cache: [props.doctype, route.query.view],
  onSuccess(data) {
    setupViews(data.views)
    setupDefaults(data)
  },
})

onMounted(() => {
  useDebounceFn(() => reload(), 100)()
})

function reload() {
  list.value.params = getParams()
  list.value.reload()
}

const viewsDropdownOptions = ref([])

const defaultViews = [
  {
    label: 'List View',
    icon: 'list',
    onClick() {
      viewUpdated.value = false
      router.push({ name: route.name })
    },
  },
]

function setupViews(views) {
  viewsDropdownOptions.value = [
    {
      group: 'Default Views',
      hideLabel: true,
      items: defaultViews,
    },
  ]

  views?.forEach((view) => {
    view.icon = view.icon || 'list'
    view.filters = JSON.parse(view.filters)
    view.onClick = () => {
      viewUpdated.value = false
      router.push({ ...route, query: { view: view.name } })
    }
  })

  let pinnedViews = views?.filter((v) => v.pinned) || []
  let savedViews = views?.filter((v) => !v.pinned) || []

  if (savedViews.length) {
    viewsDropdownOptions.value.push({
      group: 'Saved Views',
      items: savedViews,
    })
  }
  if (pinnedViews.length) {
    viewsDropdownOptions.value.push({
      group: 'Pinned Views',
      items: pinnedViews,
    })
  }
}

function setupDefaults(data) {
  let cv = getView(route.query.view)

  defaultParams.value = {
    doctype: props.doctype,
    filters: list.value.params.filters,
    order_by: list.value.params.order_by,
    columns: data.columns,
    rows: data.rows,
    custom_view_name: cv?.name || '',
  }
}

function updateFilter(filters) {
  viewUpdated.value = true
  if (!defaultParams.value) {
    defaultParams.value = getParams()
  }
  list.value.params = defaultParams.value
  list.value.params.filters = filters
  list.value.reload()
}

function updateSort(order_by) {
  viewUpdated.value = true
  if (!defaultParams.value) {
    defaultParams.value = getParams()
  }
  list.value.params = defaultParams.value
  list.value.params.order_by = order_by
  list.value.reload()
}

function updateColumns(obj) {
  defaultParams.value.columns = obj.isDefault ? '' : obj.columns
  defaultParams.value.rows = obj.isDefault ? '' : obj.rows
  view.value.default_columns = obj.isDefault

  if (obj.reset) {
    defaultParams.value.columns = getParams().columns
    defaultParams.value.rows = getParams().rows
  }

  if (obj.reload) {
    list.value.params = defaultParams.value
    list.value.reload()
  }
  viewUpdated.value = true
}

// View Actions
const viewActions = computed(() => {
  let o = [
    {
      group: 'Default Views',
      hideLabel: true,
      items: [
        {
          label: 'Duplicate',
          icon: () => h(DuplicateIcon, { class: 'h-4 w-4' }),
          onClick: () => {
            view.value.name = ''
            view.value.label = view.value.label + ' New'
            showViewModal.value = true
          },
        },
      ],
    },
  ]

  if (route.query.view) {
    o[0].items.push({
      label: view.value.pinned ? 'Unpin View' : 'Pin View',
      icon: () =>
        h(view.value.pinned ? UnpinIcon : PinIcon, { class: 'h-4 w-4' }),
      onClick: () => {
        call('crm.fcrm.doctype.crm_view_settings.crm_view_settings.pin', {
          name: route.query.view,
          value: !view.value.pinned,
        }).then(() => {
          view.value.pinned = !view.value.pinned
          reloadView()
        })
      },
    })

    o.push({
      group: 'Delete View',
      hideLabel: true,
      items: [
        {
          label: 'Delete',
          icon: 'trash-2',
          onClick: () =>
            $dialog({
              title: 'Delete View',
              message: 'Are you sure you want to delete this view?',
              variant: 'danger',
              actions: [
                {
                  label: 'Delete',
                  variant: 'solid',
                  theme: 'red',
                  onClick: (close) => {
                    close()
                    call(
                      'crm.fcrm.doctype.crm_view_settings.crm_view_settings.delete',
                      {
                        name: route.query.view,
                      }
                    ).then(() => {
                      router.push({ name: route.name })
                      reloadView()
                    })
                  },
                },
              ],
            }),
        },
      ],
    })
  }
  return o
})

function cancelChanges() {
  reload()
  viewUpdated.value = false
}

function saveView() {
  view.value = {
    label: view.value.label,
    name: view.value.name,
    filters: defaultParams.value.filters,
    order_by: defaultParams.value.order_by,
    columns: defaultParams.value.columns,
    rows: defaultParams.value.rows,
    route_name: route.name,
    default_columns: view.value.default_columns,
  }
  showViewModal.value = true
}

// Watchers
watch(
  () => getView(route.query.view),
  (value, old_value) => {
    if (JSON.stringify(value) === JSON.stringify(old_value)) return
    reload()
  },
  { deep: true }
)

watch(
  () => route,
  (value, old_value) => {
    if (value === old_value) return
    reload()
  }
)
</script>