<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { useRouter } from 'vue-router'
import type { TableColumn } from '@nuxt/ui'

import { supabase } from '../lib/supabase'
import { useAuth } from '../composables/useAuth'

type DailyDealFlow = {
  id: string
  submission_id: string
  insured_name: string | null
  client_phone_number: string | null
  lead_vendor: string | null
  date: string | null
  status: string | null
  agent: string | null
  carrier: string | null
  created_at: string | null
}

const router = useRouter()
const auth = useAuth()

const loading = ref(false)
const error = ref<string | null>(null)
const query = ref('')

const rows = ref<DailyDealFlow[]>([])

const filteredRows = computed(() => {
  const q = query.value.trim().toLowerCase()
  if (!q) return rows.value

  return rows.value.filter((r) => {
    const haystack = [
      r.submission_id,
      r.insured_name ?? '',
      r.client_phone_number ?? '',
      r.lead_vendor ?? '',
      r.status ?? '',
      r.agent ?? '',
      r.carrier ?? ''
    ].join(' ').toLowerCase()

    return haystack.includes(q)
  })
})

const columns: TableColumn<DailyDealFlow>[] = [
  {
    accessorKey: 'submission_id',
    header: 'Submission ID'
  },
  {
    accessorKey: 'insured_name',
    header: 'Insured'
  },
  {
    accessorKey: 'client_phone_number',
    header: 'Phone'
  },
  {
    accessorKey: 'lead_vendor',
    header: 'Vendor'
  },
  {
    accessorKey: 'status',
    header: 'Status'
  },
  {
    accessorKey: 'date',
    header: 'Date'
  },
  {
    id: 'actions',
    header: 'Actions',
    meta: { align: 'center', class: 'w-[110px]' }
  }
]

const load = async () => {
  loading.value = true
  error.value = null

  try {
    await auth.init()

    const isSuperAdmin = Boolean(auth.state.value.profile?.is_super_admin)
    const centerId = auth.state.value.profile?.center_id ?? null
    let leadVendor = auth.state.value.profile?.lead_vendor ?? null

    if (!isSuperAdmin && !leadVendor && centerId) {
      const { data: center, error: centerError } = await supabase
        .from('centers')
        .select('lead_vendor')
        .eq('id', centerId)
        .maybeSingle()

      if (centerError) throw centerError
      leadVendor = (center?.lead_vendor as string | null) ?? null
    }

    if (!isSuperAdmin && !leadVendor) {
      rows.value = []
      return
    }

    let q = supabase
      .from('daily_deal_flow')
      .select('id,submission_id,insured_name,client_phone_number,lead_vendor,date,status,agent,carrier,created_at')
      .order('created_at', { ascending: false })
      .limit(250)

    if (!isSuperAdmin) {
      q = q.eq('lead_vendor', leadVendor)
    }

    const { data, error: supaError } = await q

    if (supaError) throw supaError

    rows.value = (data ?? []) as DailyDealFlow[]
  } catch (e) {
    const msg = e instanceof Error ? e.message : 'Failed to load retainers'
    error.value = msg
  } finally {
    loading.value = false
  }
}

onMounted(load)

const openRow = (row: DailyDealFlow) => {
  router.push(`/retainers/${row.id}`)
}
</script>

<template>
  <UDashboardPanel id="retainers">
    <template #header>
      <UDashboardNavbar title="Retainers">
        <template #leading>
          <UDashboardSidebarCollapse />
        </template>

        <template #right>
          <UButton
            color="neutral"
            variant="outline"
            icon="i-lucide-refresh-cw"
            :loading="loading"
            @click="load"
          >
            Refresh
          </UButton>
        </template>
      </UDashboardNavbar>
    </template>

    <template #body>
      <div class="flex flex-wrap items-center justify-between gap-3">
        <UInput
          v-model="query"
          class="max-w-md"
          icon="i-lucide-search"
          placeholder="Search by submission, name, phone, vendor..."
        />

        <UBadge
          variant="subtle"
          :label="`${filteredRows.length} leads`"
        />
      </div>

      <UAlert
        v-if="error"
        class="mt-4"
        color="error"
        variant="subtle"
        title="Unable to load retainers"
        :description="error"
      />

      <UTable
        class="mt-4"
        :loading="loading"
        :data="filteredRows"
        :columns="columns"
        :ui="{
          base: 'w-full table-fixed border-separate border-spacing-0',
          thead: '[&>tr]:bg-elevated/50 [&>tr]:after:content-none',
          tbody: '[&>tr]:last:[&>td]:border-b-0',
          th: 'first:rounded-l-lg last:rounded-r-lg border-y border-default first:border-l last:border-r',
          td: 'border-b border-default'
        }"
      >
        <template #submission_id-cell="{ row }">
          <button type="button" class="block w-full text-left font-medium text-highlighted" @click="openRow(row.original)">
            {{ row.original.submission_id }}
          </button>
        </template>

        <template #insured_name-cell="{ row }">
          <button type="button" class="block w-full text-left" @click="openRow(row.original)">
            {{ row.original.insured_name ?? '—' }}
          </button>
        </template>

        <template #client_phone_number-cell="{ row }">
          <button type="button" class="block w-full text-left" @click="openRow(row.original)">
            {{ row.original.client_phone_number ?? '—' }}
          </button>
        </template>

        <template #lead_vendor-cell="{ row }">
          <button type="button" class="block w-full text-left" @click="openRow(row.original)">
            {{ row.original.lead_vendor ?? '—' }}
          </button>
        </template>

        <template #status-cell="{ row }">
          <button type="button" class="block w-full text-left" @click="openRow(row.original)">
            {{ row.original.status ?? '—' }}
          </button>
        </template>

        <template #date-cell="{ row }">
          <button type="button" class="block w-full text-left" @click="openRow(row.original)">
            {{ row.original.date ?? '—' }}
          </button>
        </template>

        <template #actions-cell="{ row }">
          <div class="flex justify-center">
            <UButton
              icon="i-lucide-eye"
              color="neutral"
              variant="outline"
              label="View"
              @click="openRow(row.original)"
            />
          </div>
        </template>
      </UTable>
    </template>
  </UDashboardPanel>
</template>
