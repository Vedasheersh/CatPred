<script setup lang="ts">
import { ref, watch } from 'vue'
import type { PredictionMode } from '../api/types'
import InputRow from './InputRow.vue'
import BatchUpload from './BatchUpload.vue'
import type { InputRowEntry } from '../composables/useInputRows'

const props = defineProps<{
  rows: InputRowEntry[]
  mode: PredictionMode
  disabled: boolean
}>()

const emit = defineEmits<{
  addRow: []
  removeRow: [id: number]
  updateField: [id: number, field: 'sequence' | 'pdbpath' | 'inhibitorSmiles', value: string]
  addSubstrate: [rowId: number]
  removeSubstrate: [rowId: number, subId: number]
  updateSubstrateSmiles: [rowId: number, subId: number, smiles: string]
  setPrimary: [rowId: number, subId: number]
  loadSample: []
  importCsv: [text: string]
  submit: []
}>()

const openRowIds = ref<Set<number>>(new Set())
let openNextAddedRow = false

function isRowOpen(id: number): boolean {
  return openRowIds.value.has(id)
}

function onEntryToggle(id: number, event: Event) {
  const target = event.target as HTMLDetailsElement
  const next = new Set(openRowIds.value)
  if (target.open) {
    next.add(id)
  } else {
    next.delete(id)
  }
  openRowIds.value = next
}

function requestAddRow() {
  openNextAddedRow = true
  emit('addRow')
}

function requestLoadSample() {
  openNextAddedRow = false
  emit('loadSample')
}

function requestImportCsv(text: string) {
  openNextAddedRow = false
  emit('importCsv', text)
}

function entryStatus(row: InputRowEntry): string {
  const hasSequence = row.sequence.trim().length > 0
  const hasId = row.pdbpath.trim().length > 0
  const hasChemistry = props.mode === 'substrate'
    ? row.substrates.some((s) => s.smiles.trim())
    : row.inhibitorSmiles.trim().length > 0

  if (hasSequence && hasId && hasChemistry) return 'ready'
  return 'incomplete'
}

watch(
  () => props.rows.map((row) => row.id),
  (ids, previousIds = []) => {
    const currentIds = new Set(ids)
    const nextOpen = new Set(
      [...openRowIds.value].filter((id) => currentIds.has(id)),
    )

    if (ids.length === 1) {
      nextOpen.add(ids[0])
    } else if (ids.length > 1) {
      const addedIds = ids.filter((id) => !previousIds.includes(id))
      if (openNextAddedRow && addedIds.length > 0) {
        for (const id of addedIds) nextOpen.add(id)
      } else if (nextOpen.size === 0) {
        nextOpen.add(ids[0])
      }
    }

    openRowIds.value = nextOpen
    openNextAddedRow = false
  },
  { immediate: true },
)
</script>

<template>
  <form
    class="input-panel"
    novalidate
    :aria-busy="disabled"
    @submit.prevent="emit('submit')"
  >
    <div class="panel-head">
      <h3 class="panel-title">Inputs</h3>
      <span class="entry-count">{{ rows.length }} {{ rows.length === 1 ? 'entry' : 'entries' }}</span>
    </div>

    <div class="rows-list">
      <template v-if="rows.length <= 1">
        <InputRow
          v-for="(row, index) in rows"
          :key="row.id"
          :row="row"
          :mode="mode"
          :index="index"
          :show-header="rows.length > 1"
          :can-remove="rows.length > 1"
          @remove="(id) => emit('removeRow', id)"
          @update-field="(id, field, value) => emit('updateField', id, field, value)"
          @add-substrate="(rowId) => emit('addSubstrate', rowId)"
          @remove-substrate="(rowId, subId) => emit('removeSubstrate', rowId, subId)"
          @update-substrate-smiles="(rowId, subId, smiles) => emit('updateSubstrateSmiles', rowId, subId, smiles)"
          @set-primary="(rowId, subId) => emit('setPrimary', rowId, subId)"
        />
      </template>

      <details
        v-else
        v-for="(row, index) in rows"
        :key="row.id"
        class="entry-shell"
        :open="isRowOpen(row.id)"
        @toggle="onEntryToggle(row.id, $event)"
      >
        <summary class="entry-summary">
          <span class="entry-name">Entry {{ index + 1 }}</span>
          <span class="entry-id">{{ row.pdbpath || 'No ID' }}</span>
          <span
            class="entry-status"
            :class="{ ready: entryStatus(row) === 'ready' }"
          >{{ entryStatus(row) }}</span>
          <button
            v-if="rows.length > 1"
            type="button"
            class="remove-entry-btn"
            :aria-label="`Remove entry ${index + 1}`"
            @click.stop.prevent="emit('removeRow', row.id)"
          >Remove</button>
        </summary>

        <InputRow
          :row="row"
          :mode="mode"
          :index="index"
          :show-header="false"
          :can-remove="false"
          @remove="(id) => emit('removeRow', id)"
          @update-field="(id, field, value) => emit('updateField', id, field, value)"
          @add-substrate="(rowId) => emit('addSubstrate', rowId)"
          @remove-substrate="(rowId, subId) => emit('removeSubstrate', rowId, subId)"
          @update-substrate-smiles="(rowId, subId, smiles) => emit('updateSubstrateSmiles', rowId, subId, smiles)"
          @set-primary="(rowId, subId) => emit('setPrimary', rowId, subId)"
        />
      </details>
    </div>

    <BatchUpload @import="requestImportCsv" />

    <div class="actions">
      <button
        type="button"
        class="btn btn-ghost"
        :disabled="disabled"
        @click="requestAddRow"
      >Add entry</button>
      <button
        type="button"
        class="btn btn-ghost"
        :disabled="disabled"
        @click="requestLoadSample"
      >Load sample</button>
      <button
        type="submit"
        class="btn btn-primary"
        :disabled="disabled"
        :aria-busy="disabled"
      >
        <span v-if="disabled" class="spinner" aria-hidden="true"></span>
        {{ disabled ? 'Running...' : 'Run prediction' }}
      </button>
    </div>

    <p v-if="mode === 'substrate'" class="helper-text">
      Add substrates per entry. The <strong>primary</strong> substrate is used for Km;
      all substrates (joined) are used for kcat.
    </p>
    <p v-else class="helper-text">
      Enter the inhibitor compound SMILES, enzyme sequence, and a Sequence ID.
    </p>
  </form>
</template>

<style scoped>
.input-panel {
  background: var(--bg-surface);
  border: 1px solid var(--border);
  border-radius: var(--radius-lg);
  padding: 1rem;
  box-shadow: var(--shadow-sm);
  min-width: 0;
}

.panel-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  margin-bottom: 0.75rem;
}

.panel-title {
  font-family: var(--font-serif);
  font-size: 1.25rem;
  font-weight: 400;
  color: var(--text);
}

.entry-count {
  flex-shrink: 0;
  font-family: var(--font-mono);
  font-size: 0.68rem;
  font-weight: 500;
  color: var(--text-tertiary);
  letter-spacing: 0.04em;
  text-transform: uppercase;
}

.rows-list {
  display: grid;
  gap: 0.5rem;
  margin-bottom: 0.75rem;
}

.entry-shell {
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
  background: var(--bg-surface);
  overflow: hidden;
}

.entry-shell[open] {
  box-shadow: var(--shadow-sm);
}

.entry-summary {
  display: grid;
  grid-template-columns: auto minmax(0, 1fr) auto auto;
  align-items: center;
  gap: 0.5rem;
  min-height: 42px;
  padding: 0.5rem 0.75rem;
  cursor: pointer;
  color: var(--text-secondary);
}

.entry-summary::marker {
  color: var(--text-tertiary);
}

.entry-name {
  font-family: var(--font-mono);
  font-size: 0.68rem;
  font-weight: 600;
  color: var(--text-secondary);
  letter-spacing: 0.04em;
  text-transform: uppercase;
}

.entry-id {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-family: var(--font-mono);
  font-size: 0.68rem;
  color: var(--text-tertiary);
}

.entry-status {
  font-family: var(--font-mono);
  font-size: 0.6rem;
  font-weight: 500;
  color: var(--text-tertiary);
  border: 1px solid var(--border);
  border-radius: 999px;
  padding: 0.075rem 0.4rem;
  text-transform: uppercase;
}

.entry-status.ready {
  color: var(--ok);
  border-color: rgba(5, 150, 105, 0.35);
  background: rgba(5, 150, 105, 0.06);
}

.remove-entry-btn {
  height: 26px;
  padding: 0 0.5rem;
  border: 1px solid var(--border);
  border-radius: 999px;
  font-family: var(--font-mono);
  font-size: 0.6rem;
  font-weight: 500;
  color: var(--danger);
  background: var(--bg-surface);
  transition: background 0.15s, border-color 0.15s;
}

.remove-entry-btn:hover {
  background: rgba(220, 38, 38, 0.06);
  border-color: rgba(220, 38, 38, 0.3);
}

.entry-shell :deep(.row-item) {
  border: 0;
  border-top: 1px solid var(--border);
  border-radius: 0;
  padding: 0.75rem;
}

.actions {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-top: 0.75rem;
}

.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.375rem;
  height: 40px;
  padding: 0 1rem;
  border: 1px solid var(--border);
  border-radius: 999px;
  font-size: 0.8rem;
  font-weight: 500;
  transition: all 0.15s;
  min-width: 44px;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-ghost {
  background: transparent;
  color: var(--text-secondary);
}

.btn-ghost:hover:not(:disabled) {
  background: var(--bg-muted);
  color: var(--text);
}

.btn-primary {
  background: var(--accent);
  border-color: var(--accent);
  color: #fff;
  box-shadow: 0 1px 3px rgba(5, 150, 105, 0.2);
}

.btn-primary:hover:not(:disabled) {
  background: var(--accent-hover);
  border-color: var(--accent-hover);
}

.spinner {
  width: 14px;
  height: 14px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top-color: #fff;
  border-radius: 50%;
  animation: spin 0.6s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.helper-text {
  margin-top: 0.75rem;
  font-size: 0.8rem;
  color: var(--text-tertiary);
  line-height: 1.5;
}

.helper-text strong {
  font-weight: 600;
  color: var(--text-secondary);
}

@media (max-width: 520px) {
  .entry-summary {
    grid-template-columns: auto minmax(0, 1fr) auto;
  }

  .entry-status {
    display: none;
  }
}
</style>
