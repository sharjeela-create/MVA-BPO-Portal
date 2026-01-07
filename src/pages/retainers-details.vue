<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { useRoute, useRouter } from 'vue-router'

import { supabase } from '../lib/supabase'
import { useAuth } from '../composables/useAuth'

type DailyDealFlow = Record<string, unknown> & {
  id: string
  submission_id: string
  insured_name?: string | null
  client_phone_number?: string | null
  created_at?: string | null
  updated_at?: string | null
}

type CallResult = {
  id: string
  submission_id: string
  application_submitted: boolean
  status: string | null
  notes: string | null
  carrier: string | null
  product_type: string | null
  draft_date: string | null
  new_draft_date: string | null
  submitting_agent: string | null
  agent_who_took_call: string | null
  call_source: string | null
  dq_reason: string | null
  monthly_premium: number | string | null
  coverage_amount: number | string | null
  face_amount: number | string | null
  submission_date: string | null
  created_at: string | null
  updated_at?: string | null
}

const route = useRoute()
const router = useRouter()
const auth = useAuth()

const id = computed(() => route.params.id as string)

const loading = ref(false)
const error = ref<string | null>(null)
const row = ref<DailyDealFlow | null>(null)
const callResults = ref<CallResult[]>([])
const activeTab = ref('basic')

const tabs = [
  { label: 'Basic Information', icon: 'i-lucide-user', value: 'basic' },
  { label: 'Accident Details', icon: 'i-lucide-car', value: 'accident' },
  { label: 'Insurance & Policy', icon: 'i-lucide-shield', value: 'insurance' },
  { label: 'Additional Info', icon: 'i-lucide-info', value: 'additional' },
  { label: 'Call Updates', icon: 'i-lucide-phone', value: 'calls' }
]

const headerTitle = computed(() => {
  if (!row.value) return 'Lead details'
  const name = row.value.insured_name || 'Unknown'
  const phone = row.value.client_phone_number || 'N/A'
  return `${name} - ${phone}`
})

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
      error.value = 'Lead not found'
      row.value = null
      return
    }

    let q = supabase
      .from('daily_deal_flow')
      .select('*')
      .eq('id', id.value)

    if (!isSuperAdmin) {
      q = q.eq('lead_vendor', leadVendor)
    }

    const { data, error: supaError } = await q.maybeSingle()

    if (supaError) throw supaError
    if (!data) {
      error.value = 'Lead not found'
      row.value = null
      return
    }

    row.value = data as DailyDealFlow
    console.log('✅ Lead data loaded:', row.value)
    console.log('📋 All fields:', Object.keys(row.value))

    // Load call results by submission_id
    const { data: results, error: callResultsError } = await supabase
      .from('call_results')
      .select('id,submission_id,application_submitted,status,notes,carrier,product_type,draft_date,new_draft_date,submitting_agent,agent_who_took_call,call_source,dq_reason,monthly_premium,coverage_amount,face_amount,submission_date,created_at,updated_at')
      .eq('submission_id', row.value.submission_id)
      .order('created_at', { ascending: false })

    if (callResultsError) throw callResultsError

    callResults.value = (results ?? []) as CallResult[]
    console.log('📞 Call results loaded:', callResults.value.length, 'records')
    if (callResults.value[0]) {
      console.log('📞 Call result fields:', Object.keys(callResults.value[0] as Record<string, unknown>))
    }
  } catch (e) {
    const msg = e instanceof Error ? e.message : 'Failed to load lead'
    error.value = msg
  } finally {
    loading.value = false
  }
}

onMounted(load)

const basicInfoFields = computed(() => {
  if (!row.value) return []
  const fields = [
    ['insured_name', 'Customer Name'],
    ['client_phone_number', 'Phone Number'],
    ['contact_name', 'Contact Name'],
    ['contact_number', 'Contact Number'],
    ['contact_address', 'Contact Address'],
    ['status', 'Status'],
    ['lead_vendor', 'Lead Vendor'],
    ['agent', 'Agent'],
    ['date', 'Date']
  ].map(([key, label]) => ({ key, label, value: (row.value as any)[key] }))
  console.log('👤 Basic Info Fields:', fields)
  return fields
})

const accidentDetailsFields = computed(() => {
  if (!row.value) return []
  const fields = [
    ['accident_date', 'Accident Date'],
    ['accident_location', 'Accident Location'],
    ['accident_scenario', 'Accident Scenario'],
    ['prior_attorney_involved', 'Prior Attorney Involved'],
    ['prior_attorney_details', 'Prior Attorney Details'],
    ['medical_attention', 'Medical Attention'],
    ['police_attended', 'Police Attended'],
    ['injuries', 'Injuries'],
    ['other_party_admit_fault', 'Other Party Admit Fault'],
    ['passengers_count', 'Passengers Count']
  ].map(([key, label]) => ({ key, label, value: (row.value as any)[key] }))
  console.log('🚗 Accident Details Fields:', fields)
  return fields
})

const insurancePolicyFields = computed(() => {
  if (!row.value) return []
  const fields = [
    ['insured', 'Insured'],
    ['insurance_company', 'Insurance Company'],
    ['vehicle_registration', 'Vehicle Registration'],
    ['third_party_vehicle_registration', 'Third Party Vehicle Registration'],
    ['carrier', 'Carrier'],
    ['carrier_attempted_1', 'Carrier Attempted 1'],
    ['carrier_attempted_2', 'Carrier Attempted 2'],
    ['carrier_attempted_3', 'Carrier Attempted 3'],
    ['product_type', 'Product Type'],
    ['policy_number', 'Policy Number'],
    ['monthly_premium', 'Monthly Premium'],
    ['face_amount', 'Face Amount'],
    ['placement_status', 'Placement Status']
  ].map(([key, label]) => ({ key, label, value: (row.value as any)[key] }))
  console.log('🛡️ Insurance Policy Fields:', fields)
  return fields
})

const additionalInfoFields = computed(() => {
  if (!row.value) return []
  const fields = [
    ['buffer_agent', 'Buffer Agent'],
    ['licensed_agent_account', 'Licensed Agent Account'],
    ['call_result', 'Call Result'],
    ['draft_date', 'Draft Date'],
    ['from_callback', 'From Callback'],
    ['is_callback', 'Is Callback'],
    ['is_retention_call', 'Is Retention Call'],
    ['carrier_audit', 'Carrier Audit'],
    ['product_type_carrier', 'Product Type Carrier'],
    ['level_or_gi', 'Level or GI'],
    ['notes', 'Notes'],
    ['ghl_location_id', 'GHL Location ID'],
    ['ghl_opportunity_id', 'GHL Opportunity ID'],
    ['ghlcontactid', 'GHL Contact ID'],
    ['sync_status', 'Sync Status'],
    ['created_at', 'Created At'],
    ['updated_at', 'Updated At']
  ].map(([key, label]) => ({ key, label, value: (row.value as any)[key] }))
  console.log('ℹ️ Additional Info Fields:', fields)
  return fields
})

function formatValue(value: unknown) {
  if (value === null || value === undefined || value === '') return '—'
  if (typeof value === 'boolean') return value ? 'Yes' : 'No'
  if (typeof value === 'number') return String(value)
  if (typeof value === 'string') return value
  return JSON.stringify(value)
}

function formatDateTime(value: string | null | undefined) {
  if (!value) return '—'
  const d = new Date(value)
  if (Number.isNaN(d.getTime())) return value
  return d.toLocaleString('en-US', {
    year: 'numeric',
    month: 'short',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit'
  })
}

function formatDateOnly(value: string | null | undefined) {
  if (!value) return '—'
  const d = new Date(value)
  if (Number.isNaN(d.getTime())) return value
  return d.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'short',
    day: '2-digit'
  })
}
</script>

<template>
  <UDashboardPanel id="retainer-details">
    <template #header>
      <UDashboardNavbar :title="headerTitle">
        <template #leading>
          <UButton
            color="neutral"
            variant="ghost"
            icon="i-lucide-arrow-left"
            @click="router.push('/retainers')"
          >
            Back
          </UButton>
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
      <UAlert
        v-if="error"
        color="error"
        variant="subtle"
        title="Unable to load lead"
        :description="error"
      />

      <div v-else-if="loading" class="flex h-full min-h-64 items-center justify-center">
        <UIcon name="i-lucide-loader-circle" class="size-8 animate-spin text-dimmed" />
      </div>

      <div v-else-if="row" class="space-y-4">
        <UTabs v-model="activeTab" :items="tabs">
          <template #content="{ item }">
            <UCard v-if="item.value === 'basic'">
              <div class="grid gap-4 md:grid-cols-2">
                <div
                  v-for="field in basicInfoFields"
                  :key="field.key"
                  class="rounded-lg border border-default bg-elevated/20 p-3"
                >
                  <div class="text-xs uppercase tracking-wide text-muted">
                    {{ field.label }}
                  </div>
                  <div class="mt-1 text-sm text-highlighted wrap-break-word">
                    {{ formatValue(field.value) }}
                  </div>
                </div>
              </div>
            </UCard>

            <UCard v-else-if="item.value === 'accident'">
              <div class="grid gap-4 md:grid-cols-2">
                <div
                  v-for="field in accidentDetailsFields"
                  :key="field.key"
                  class="rounded-lg border border-default bg-elevated/20 p-3"
                >
                  <div class="text-xs uppercase tracking-wide text-muted">
                    {{ field.label }}
                  </div>
                  <div class="mt-1 text-sm text-highlighted wrap-break-word">
                    {{ formatValue(field.value) }}
                  </div>
                </div>
              </div>
            </UCard>

            <UCard v-else-if="item.value === 'insurance'">
              <div class="grid gap-4 md:grid-cols-2">
                <div
                  v-for="field in insurancePolicyFields"
                  :key="field.key"
                  class="rounded-lg border border-default bg-elevated/20 p-3"
                >
                  <div class="text-xs uppercase tracking-wide text-muted">
                    {{ field.label }}
                  </div>
                  <div class="mt-1 text-sm text-highlighted wrap-break-word">
                    {{ formatValue(field.value) }}
                  </div>
                </div>
              </div>
            </UCard>

            <UCard v-else-if="item.value === 'additional'">
              <div class="grid gap-4 md:grid-cols-2">
                <div
                  v-for="field in additionalInfoFields"
                  :key="field.key"
                  class="rounded-lg border border-default bg-elevated/20 p-3"
                >
                  <div class="text-xs uppercase tracking-wide text-muted">
                    {{ field.label }}
                  </div>
                  <div class="mt-1 text-sm text-highlighted wrap-break-word">
                    {{ formatValue(field.value) }}
                  </div>
                </div>
              </div>
            </UCard>

            <UCard v-else-if="item.value === 'calls'">
              <div v-if="callResults.length === 0" class="py-8 text-center text-muted">
                No call results found
              </div>

              <div v-else class="space-y-4">
                <div
                  v-for="result in callResults"
                  :key="result.id"
                  class="rounded-lg border border-default bg-elevated/20 p-4"
                >
                  <div class="flex items-start justify-between gap-3 mb-3">
                    <div>
                      <div class="font-medium text-highlighted">
                        {{ result.agent_who_took_call ?? result.submitting_agent ?? 'Call Result' }}
                      </div>
                      <div class="text-xs text-muted mt-1">
                        {{ formatDateTime(result.created_at) }}
                      </div>
                    </div>

                    <div class="flex items-center gap-2 shrink-0">
                      <UBadge
                        variant="subtle"
                        :color="result.application_submitted ? 'success' : 'neutral'"
                        :label="result.application_submitted ? 'Application Submitted' : 'Not Submitted'"
                      />
                      <UBadge
                        v-if="result.status"
                        variant="subtle"
                        :label="result.status"
                      />
                    </div>
                  </div>

                  <div class="grid gap-3 md:grid-cols-2">
                    <div class="rounded-lg border border-default bg-elevated/10 p-3">
                      <div class="text-xs uppercase tracking-wide text-muted">Submission Date</div>
                      <div class="mt-1 text-sm text-highlighted">{{ formatDateOnly(result.submission_date) }}</div>
                    </div>

                    <div class="rounded-lg border border-default bg-elevated/10 p-3">
                      <div class="text-xs uppercase tracking-wide text-muted">Call Source</div>
                      <div class="mt-1 text-sm text-highlighted">{{ formatValue(result.call_source) }}</div>
                    </div>

                    <div class="rounded-lg border border-default bg-elevated/10 p-3">
                      <div class="text-xs uppercase tracking-wide text-muted">Carrier</div>
                      <div class="mt-1 text-sm text-highlighted">{{ formatValue(result.carrier) }}</div>
                    </div>

                    <div class="rounded-lg border border-default bg-elevated/10 p-3">
                      <div class="text-xs uppercase tracking-wide text-muted">Product Type</div>
                      <div class="mt-1 text-sm text-highlighted">{{ formatValue(result.product_type) }}</div>
                    </div>

                    <div class="rounded-lg border border-default bg-elevated/10 p-3">
                      <div class="text-xs uppercase tracking-wide text-muted">Draft Date</div>
                      <div class="mt-1 text-sm text-highlighted">{{ formatDateOnly(result.new_draft_date || result.draft_date) }}</div>
                    </div>

                    <div class="rounded-lg border border-default bg-elevated/10 p-3">
                      <div class="text-xs uppercase tracking-wide text-muted">Monthly Premium</div>
                      <div class="mt-1 text-sm text-highlighted">{{ formatValue(result.monthly_premium) }}</div>
                    </div>
                  </div>

                  <div class="mt-3 rounded-lg border border-default bg-elevated/10 p-3">
                    <div class="text-xs uppercase tracking-wide text-muted">Notes</div>
                    <div v-if="result.notes" class="mt-1 text-sm text-highlighted wrap-break-word">
                      {{ result.notes }}
                    </div>
                    <div v-else class="mt-1 text-sm text-muted italic">No notes</div>
                  </div>
                </div>
              </div>
            </UCard>
          </template>
        </UTabs>
      </div>
    </template>
  </UDashboardPanel>
</template>
