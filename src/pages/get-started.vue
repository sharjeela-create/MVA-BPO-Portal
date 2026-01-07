<script setup lang="ts">
import { ref } from 'vue'
import { RouterLink } from 'vue-router'
import { z } from 'zod'

import { BRANDING } from '../lib/branding'
import { supabase } from '../lib/supabase'

const schema = z.object({
  publisherName: z.string().trim().min(2, 'Enter the publisher name'),
  publisherEmail: z.string().trim().email('Enter a valid publisher email'),
  adminName: z.string().trim().min(2, 'Enter the admin name'),
  adminEmail: z.string().trim().email('Enter a valid admin email')
})

type FormState = z.infer<typeof schema>

const form = ref<FormState>({
  publisherName: '',
  publisherEmail: '',
  adminName: '',
  adminEmail: ''
})

const fieldErrors = ref<Partial<Record<keyof FormState, string>>>({})
const formError = ref<string | null>(null)
const isSubmitting = ref(false)
const isSuccess = ref(false)

const validate = () => {
  fieldErrors.value = {}
  formError.value = null

  const result = schema.safeParse(form.value)
  if (result.success) return true

  const issues = result.error.issues
  for (const issue of issues) {
    const key = issue.path[0] as keyof FormState | undefined
    if (!key) continue
    if (!fieldErrors.value[key]) fieldErrors.value[key] = issue.message
  }

  return false
}

const handleSubmit = async () => {
  if (isSubmitting.value) return

  const ok = validate()
  if (!ok) return

  isSubmitting.value = true
  formError.value = null

  try {
    const { error } = await supabase
      .from('publisher_onboarding')
      .insert({
        publisher_name: form.value.publisherName.trim(),
        publisher_email: form.value.publisherEmail.trim().toLowerCase(),
        admin_name: form.value.adminName.trim(),
        admin_email: form.value.adminEmail.trim().toLowerCase()
      })

    if (error) throw error

    isSuccess.value = true
  } catch (err) {
    const msg = err instanceof Error ? err.message : 'Unable to submit your details'
    formError.value = msg
  } finally {
    isSubmitting.value = false
  }
}

const reset = () => {
  isSuccess.value = false
  formError.value = null
  fieldErrors.value = {}
  form.value = {
    publisherName: '',
    publisherEmail: '',
    adminName: '',
    adminEmail: ''
  }
}
</script>

<template>
  <div class="min-h-screen bg-[var(--ap-bg)] text-white">
    <div class="relative isolate overflow-hidden bg-gradient-to-b from-[var(--ap-surface)] via-[var(--ap-surface-soft)] to-[var(--ap-bg)]">
      <div class="absolute inset-0 pointer-events-none">
        <div class="absolute -top-32 left-1/2 h-72 w-72 -translate-x-1/2 rounded-full bg-[var(--ap-accent)]/40 blur-[120px]" />
        <div class="absolute top-16 right-8 h-56 w-56 rounded-full bg-[var(--ap-amber-soft)] blur-[110px]" />
      </div>

      <header class="flex w-full items-center justify-between px-6 py-6 lg:px-10">
        <RouterLink to="/" class="flex items-center gap-3">
          <img :src="BRANDING.logo" alt="Accident Payments" class="h-10 w-auto" />
        </RouterLink>

        <div class="flex items-center gap-3 text-sm font-medium text-white/70">
          <RouterLink to="/login" class="rounded-full border border-white/15 px-4 py-2 transition hover:border-white/40 hover:text-white">
            Log in
          </RouterLink>
        </div>
      </header>

      <main class="w-full px-6 lg:px-10">
        <section class="grid gap-10 lg:grid-cols-[1.1fr_0.9fr] lg:items-start">
          <div class="space-y-8">
            <div class="space-y-4">
              <p class="text-xs uppercase tracking-[0.4em] text-white/50">Get started</p>
              <h1 class="text-4xl font-semibold leading-tight text-white md:text-5xl">
                Tell us about your publishing operation. We’ll tailor your workspace.
              </h1>
              <p class="text-base leading-relaxed text-white/70">
                Accident Payments helps publishers and BPO teams move faster from first contact to verified handoff—without losing
                the details that keep operations compliant and consistent.
              </p>
            </div>

            <div class="grid gap-4 sm:grid-cols-2">
              <div class="rounded-3xl border border-white/10 bg-white/5 p-5">
                <p class="text-xs uppercase tracking-[0.4em] text-white/50">Why this form</p>
                <p class="mt-3 text-sm leading-relaxed text-white/70">
                  We use these details to provision your Publisher Portal workspace and assign the right onboarding flow.
                </p>
              </div>
              <div class="rounded-3xl border border-white/10 bg-white/5 p-5">
                <p class="text-xs uppercase tracking-[0.4em] text-white/50">What happens next</p>
                <p class="mt-3 text-sm leading-relaxed text-white/70">
                  You’ll get a short follow-up email with recommended workflows, playbooks, and an invite to your workspace.
                </p>
              </div>
            </div>

            <div class="rounded-3xl border border-white/10 bg-black/20 p-6">
              <h2 class="text-lg font-semibold text-white">Accident Payments is built for publisher operations</h2>
              <div class="mt-4 grid gap-3">
                <div class="flex items-start gap-3 rounded-2xl border border-white/10 bg-white/5 p-4">
                  <div class="mt-1 size-2 rounded-full bg-[var(--ap-highlight)]" />
                  <div>
                    <p class="text-sm font-semibold text-white">Keep every lead audit-ready</p>
                    <p class="mt-1 text-sm text-white/70">Traceable intake, document trails, and structured status updates.</p>
                  </div>
                </div>
                <div class="flex items-start gap-3 rounded-2xl border border-white/10 bg-white/5 p-4">
                  <div class="mt-1 size-2 rounded-full bg-[var(--ap-highlight)]" />
                  <div>
                    <p class="text-sm font-semibold text-white">Standardize your intake playbook</p>
                    <p class="mt-1 text-sm text-white/70">Scripts, QA checks, and packaging steps tuned to MVA workflows.</p>
                  </div>
                </div>
                <div class="flex items-start gap-3 rounded-2xl border border-white/10 bg-white/5 p-4">
                  <div class="mt-1 size-2 rounded-full bg-[var(--ap-highlight)]" />
                  <div>
                    <p class="text-sm font-semibold text-white">Collaborate without email threads</p>
                    <p class="mt-1 text-sm text-white/70">Role-based access for admins, agents, and partner organizations.</p>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div class="rounded-[32px] border border-white/10 bg-white/10 p-8 shadow-2xl shadow-black/50 backdrop-blur-xl">
            <div v-if="isSuccess" class="space-y-6">
              <div>
                <p class="text-xs uppercase tracking-[0.4em] text-white/60">Submitted</p>
                <h2 class="mt-2 text-3xl font-semibold text-white">Thanks — we’ll reach out shortly.</h2>
                <p class="mt-3 text-sm leading-relaxed text-white/70">
                  We’ve received your details. Keep an eye on your inbox for next steps.
                </p>
              </div>

              <div class="rounded-3xl border border-white/10 bg-black/20 p-5">
                <p class="text-xs uppercase tracking-[0.4em] text-white/50">What you shared</p>
                <dl class="mt-3 space-y-2 text-sm">
                  <div class="flex items-start justify-between gap-4">
                    <dt class="text-white/60">Publisher</dt>
                    <dd class="text-right text-white">{{ form.publisherName }}</dd>
                  </div>
                  <div class="flex items-start justify-between gap-4">
                    <dt class="text-white/60">Publisher email</dt>
                    <dd class="text-right text-white">{{ form.publisherEmail }}</dd>
                  </div>
                  <div class="flex items-start justify-between gap-4">
                    <dt class="text-white/60">Admin</dt>
                    <dd class="text-right text-white">{{ form.adminName }}</dd>
                  </div>
                  <div class="flex items-start justify-between gap-4">
                    <dt class="text-white/60">Admin email</dt>
                    <dd class="text-right text-white">{{ form.adminEmail }}</dd>
                  </div>
                </dl>
              </div>

              <div class="flex flex-col gap-3 sm:flex-row">
                <RouterLink
                  to="/"
                  class="flex-1 rounded-2xl border border-white/15 px-4 py-3 text-center text-sm font-semibold uppercase tracking-wide text-white/80 transition hover:border-white/40 hover:text-white"
                >
                  Back to home
                </RouterLink>
                <button
                  type="button"
                  class="flex-1 rounded-2xl bg-[var(--ap-accent)] px-4 py-3 text-sm font-semibold uppercase tracking-wide text-white shadow-lg"
                  style="box-shadow: 0 12px 24px var(--ap-accent-shadow);"
                  @click="reset"
                >
                  Submit another
                </button>
              </div>
            </div>

            <div v-else class="space-y-6">
              <div>
                <p class="text-xs uppercase tracking-[0.4em] text-white/60">Onboarding</p>
                <h2 class="mt-2 text-3xl font-semibold text-white">Request a workspace</h2>
                <p class="mt-3 text-sm leading-relaxed text-white/70">
                  Share a few details so we can set you up with the right templates and workflows.
                </p>
              </div>

              <form class="space-y-4" @submit.prevent="handleSubmit">
                <label class="block space-y-2 text-sm">
                  <span class="text-white/80">Publisher name <span class="text-red-400">*</span></span>
                  <input
                    v-model="form.publisherName"
                    type="text"
                    placeholder="Acme BPO"
                    autocomplete="organization"
                    class="w-full rounded-2xl border border-white/15 bg-white/5 px-4 py-3 text-white placeholder:text-white/40 focus:border-white/40 focus:outline-none"
                  >
                  <span v-if="fieldErrors.publisherName" class="text-xs text-red-100">{{ fieldErrors.publisherName }}</span>
                </label>

                <label class="block space-y-2 text-sm">
                  <span class="text-white/80">Publisher email <span class="text-red-400">*</span></span>
                  <input
                    v-model="form.publisherEmail"
                    type="email"
                    placeholder="publisher@acmebpo.com"
                    autocomplete="email"
                    class="w-full rounded-2xl border border-white/15 bg-white/5 px-4 py-3 text-white placeholder:text-white/40 focus:border-white/40 focus:outline-none"
                  >
                  <span v-if="fieldErrors.publisherEmail" class="text-xs text-red-100">{{ fieldErrors.publisherEmail }}</span>
                </label>

                <label class="block space-y-2 text-sm">
                  <span class="text-white/80">Admin name <span class="text-red-400">*</span></span>
                  <input
                    v-model="form.adminName"
                    type="text"
                    placeholder="Jane Doe"
                    autocomplete="name"
                    class="w-full rounded-2xl border border-white/15 bg-white/5 px-4 py-3 text-white placeholder:text-white/40 focus:border-white/40 focus:outline-none"
                  >
                  <span v-if="fieldErrors.adminName" class="text-xs text-red-100">{{ fieldErrors.adminName }}</span>
                </label>

                <label class="block space-y-2 text-sm">
                  <span class="text-white/80">Admin email <span class="text-red-400">*</span></span>
                  <input
                    v-model="form.adminEmail"
                    type="email"
                    placeholder="admin@acmebpo.com"
                    autocomplete="email"
                    class="w-full rounded-2xl border border-white/15 bg-white/5 px-4 py-3 text-white placeholder:text-white/40 focus:border-white/40 focus:outline-none"
                  >
                  <span v-if="fieldErrors.adminEmail" class="text-xs text-red-100">{{ fieldErrors.adminEmail }}</span>
                </label>

                <p v-if="formError" class="rounded-2xl border border-red-500/30 bg-red-500/10 px-4 py-3 text-sm text-red-100">
                  {{ formError }}
                </p>

                <button
                  type="submit"
                  class="w-full rounded-2xl bg-[var(--ap-accent)] px-4 py-3 text-sm font-semibold uppercase tracking-wide text-white shadow-lg disabled:cursor-not-allowed disabled:opacity-70"
                  style="box-shadow: 0 12px 24px var(--ap-accent-shadow);"
                  :disabled="isSubmitting"
                >
                  {{ isSubmitting ? 'Submitting…' : 'Get started' }}
                </button>

                <p class="text-xs leading-relaxed text-white/50">
                  By submitting, you agree to receive onboarding messages from Accident Payments. We do not sell your data.
                </p>
              </form>
            </div>
          </div>
        </section>
      </main>
    </div>
  </div>
</template>
