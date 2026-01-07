<script setup lang="ts">
import * as z from 'zod'
import { computed, onMounted, reactive, ref } from 'vue'
import type { FormSubmitEvent } from '@nuxt/ui'

import { useAuth } from '../../composables/useAuth'
import { supabase } from '../../lib/supabase'

const profileSchema = z.object({
  centerName: z.string().trim().min(2, 'BPO / Center name is required'),
  location: z.string().trim().optional(),
  websiteOrLinkedIn: z.string().trim().optional(),
  contactEmail: z.string().trim().email('Invalid email').optional().or(z.literal('')),
  contactPhone: z.string().trim().optional(),
  numberOfAgents: z.string().trim().optional(),
  languages: z.array(z.string()).optional(),
  operatingHours: z.string().trim().optional()
})

type ProfileSchema = z.output<typeof profileSchema>

const auth = useAuth()
const toast = useToast()

const saving = ref(false)
const loading = ref(false)
const isEditing = ref(false)

const disabled = computed(() => !isEditing.value)
const centerId = computed(() => auth.state.value.profile?.center_id ?? '')

const languageOptions = [
  'English',
  'Spanish',
  'French',
  'Arabic',
  'Portuguese',
  'German',
  'Tagalog',
  'Hindi',
  'Urdu'
]

const profile = reactive<Partial<ProfileSchema>>({
  centerName: '',
  location: '',
  websiteOrLinkedIn: '',
  contactEmail: '',
  contactPhone: '',
  numberOfAgents: '',
  languages: [],
  operatingHours: ''
})

const hydrateFromDb = (data?: any) => {
  profile.centerName = data?.center_name ?? ''
  profile.location = data?.location ?? ''
  profile.websiteOrLinkedIn = data?.website_or_linkedin ?? ''
  profile.contactEmail = data?.contact_email ?? ''
  profile.contactPhone = data?.contact_phone ?? ''
  profile.numberOfAgents = data?.number_of_agents ?? ''
  profile.languages = data?.languages ?? []
  profile.operatingHours = data?.operating_hours ?? ''
}

const loadProfile = async () => {
  if (!centerId.value) return

  loading.value = true
  try {
    const { data, error } = await supabase
      .from('bpo_profiles')
      .select('*')
      .eq('center_id', centerId.value)
      .maybeSingle()

    if (error) throw error

    hydrateFromDb(data)
  } catch (err) {
    const msg = err instanceof Error ? err.message : 'Unable to load BPO profile'
    toast.add({
      title: 'Error',
      description: msg,
      icon: 'i-lucide-x',
      color: 'error'
    })
  } finally {
    loading.value = false
  }
}

onMounted(async () => {
  await auth.init()
  await loadProfile()
})

async function onSubmit(event: FormSubmitEvent<ProfileSchema>) {
  if (!centerId.value) return

  saving.value = true
  try {
    const payload = {
      center_id: centerId.value,
      center_name: event.data.centerName.trim(),
      location: event.data.location?.trim() || null,
      website_or_linkedin: event.data.websiteOrLinkedIn?.trim() || null,
      contact_email: event.data.contactEmail?.trim() || null,
      contact_phone: event.data.contactPhone?.trim() || null,
      number_of_agents: event.data.numberOfAgents?.trim() || null,
      languages: event.data.languages ?? [],
      operating_hours: event.data.operatingHours?.trim() || null
    }

    const { error, data } = await supabase
      .from('bpo_profiles')
      .upsert(payload, { onConflict: 'center_id' })
      .select('*')
      .single()

    if (error) throw error

    hydrateFromDb(data)

    toast.add({
      title: 'Success',
      description: 'Your BPO profile has been updated.',
      icon: 'i-lucide-check',
      color: 'success'
    })

    isEditing.value = false
  } catch (err) {
    const msg = err instanceof Error ? err.message : 'Unable to update BPO profile'
    toast.add({
      title: 'Error',
      description: msg,
      icon: 'i-lucide-x',
      color: 'error'
    })
  } finally {
    saving.value = false
  }
}

const startEditing = () => {
  isEditing.value = true
}

const cancelEditing = async () => {
  isEditing.value = false
  await loadProfile()
}
</script>

<template>
  <UForm
    id="bpo-profile"
    :schema="profileSchema"
    :state="profile"
    @submit="onSubmit"
  >
    <UPageCard
      title="BPO Profile"
      description="Tell us about your center so we can tailor workflows and reporting."
      variant="naked"
      orientation="horizontal"
      class="mb-4"
    >
      <div class="flex items-center gap-2 w-fit lg:ms-auto">
        <UButton
          v-if="!isEditing"
          label="Edit"
          color="neutral"
          variant="outline"
          class="w-fit"
          :loading="loading"
          @click="startEditing"
        />
        <template v-else>
          <UButton
            form="bpo-profile"
            label="Save changes"
            color="neutral"
            type="submit"
            :loading="saving"
            class="w-fit"
          />
          <UButton
            label="Cancel"
            color="neutral"
            variant="ghost"
            class="w-fit"
            :disabled="saving"
            @click="cancelEditing"
          />
        </template>
      </div>
    </UPageCard>

    <UPageCard variant="subtle" class="mb-6">
      <div class="space-y-6">
        <div class="flex items-center justify-between">
          <h3 class="text-lg font-semibold">Basic identity</h3>
        </div>

        <UFormField
          name="centerName"
          label="BPO / Center name"
          description="This is the only required field."
          required
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInput
            v-model="profile.centerName"
            placeholder="Acme BPO"
            autocomplete="organization"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="location"
          label="Location"
          description="City, Country (optional)"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInput
            v-model="profile.location"
            placeholder="Manila, Philippines"
            autocomplete="off"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="websiteOrLinkedIn"
          label="Website / LinkedIn"
          description="Optional"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInput
            v-model="profile.websiteOrLinkedIn"
            placeholder="https://www.example.com"
            autocomplete="off"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="contactEmail"
          label="Contact email"
          description="Optional"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInput
            v-model="profile.contactEmail"
            type="email"
            placeholder="ops@acmebpo.com"
            autocomplete="email"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="contactPhone"
          label="Contact phone"
          description="Optional"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInput
            v-model="profile.contactPhone"
            placeholder="+1 (555) 123-4567"
            autocomplete="tel"
            :disabled="disabled"
          />
        </UFormField>
      </div>
    </UPageCard>

    <UPageCard variant="subtle">
      <div class="space-y-6">
        <div class="flex items-center justify-between">
          <h3 class="text-lg font-semibold">Stats & capacity</h3>
        </div>

        <UFormField
          name="numberOfAgents"
          label="Number of agents"
          description="Optional"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInput
            v-model="profile.numberOfAgents"
            placeholder="e.g., 10, 50, 200+"
            autocomplete="off"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="languages"
          label="Languages"
          description="Optional"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInputMenu
            v-model="profile.languages"
            :items="languageOptions"
            multiple
            searchable
            creatable
            placeholder="Select or type languages"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="operatingHours"
          label="Operating hours"
          description="Optional"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInput
            v-model="profile.operatingHours"
            placeholder="e.g., 24/7 or 9-5 EST"
            autocomplete="off"
            :disabled="disabled"
          />
        </UFormField>
      </div>
    </UPageCard>
  </UForm>
</template>
