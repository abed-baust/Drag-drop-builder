
<script setup>
import { reactive, ref, watch, computed } from 'vue'
import Sidebar from './components/Sidebar.vue'
import Canvas from './components/Canvas.vue'
import Inspector from './components/Inspector.vue'
import PreviewModal from './components/PreviewModal.vue'
import JsonPanel from './components/JsonPanel.vue'
import { downloadJson } from './utils/download.js'

// Migrate any old schemas in localStorage (with left/right or both) to fields[]
function migrateSchema(obj){
  if(!obj || !Array.isArray(obj.steps)) return obj
  obj.steps.forEach(s => {
    if(Array.isArray(s.fields)) return
    const left = Array.isArray(s.left) ? s.left : []
    const right = Array.isArray(s.right) ? s.right : []
    s.fields = [...left, ...right]
    delete s.left; delete s.right
  })
  return obj
}

const DEFAULT_STEPS = ['Survey','Registration','Slide','Guest Ticket','Photography','Name Announcement']
const stored = localStorage.getItem('wiz-pro-single-schema')
const initial = stored ? migrateSchema(JSON.parse(stored)) : {
  steps: DEFAULT_STEPS.map((t,i)=>({ id: t.toLowerCase().replace(/\s+/g,'-'), title: t, fields: [], settings: { published: i===0, surveyParticipantType: true, publishStart: '', publishEnd: '' }, checklist:{ items:[{text:'Event creation', done:false}], published: i===0 } }))
}

const schema = reactive(initial)
const currentStepIndex = ref(0)
const selected = ref({ step: 0, index: -1 })

watch(schema, () => localStorage.setItem('wiz-pro-single-schema', JSON.stringify(schema)), { deep: true })

function newForm(){ localStorage.removeItem('wiz-pro-single-schema'); location.reload() }
function saveForm(){ const data = JSON.stringify(schema, null, 2); console.log('WIZARD JSON ->\n', data); downloadJson('wizard-schema.json', data); openJson.value = true }
function loadForm(e){
  const f = e.target.files[0]; if(!f) return
  const r = new FileReader()
  r.onload = () => { try{ const obj = migrateSchema(JSON.parse(r.result)); Object.assign(schema, obj) } catch{ alert('Invalid JSON') } }
  r.readAsText(f); e.target.value=''
}

const openPreview = ref(false)
const openJson = ref(false)
const step = computed(()=> schema.steps[currentStepIndex.value])
</script>

<template>
  <div class="topbar">
    <div class="title">Abed's Project</div>
    <div class="cta-row">
      <button class="btn" @click="newForm">New</button>
      <label class="btn" style="cursor:pointer">
        Load JSON <input type="file" accept="application/json" @change="loadForm" style="display:none">
      </label>
      <button class="btn" @click="openJson = true">Print JSON</button>
      <button class="btn primary" @click="saveForm">Save</button>
      <button class="btn" @click="openPreview = true">Preview</button>
    </div>
  </div>

  <div style="padding:12px">
    <div class="stepperTop">
      <div v-for="(s,i) in schema.steps" :key="s.id" class="stepDot" :class="{active: i===currentStepIndex}">
        <div style="width:10px;height:10px;border-radius:999px" :style="{background: i===currentStepIndex ? '#0ea5b7' : '#e5e7eb'}"></div>
        <button class="btn" @click="currentStepIndex = i">{{ s.title }}</button>
      </div>
    </div>
  </div>

  <div class="layout">
    <Sidebar :schema="schema" v-model:index="currentStepIndex" />

    <div class="panel">
      <h3>{{ step.title }} — Canvas</h3>
      <div class="body canvasWrap">
        <Canvas :step="step" :stepIndex="currentStepIndex" v-model:selected="selected" />
        <div class="panel">
          <h3>Inspector</h3>
          <div class="body">
            <Inspector :schema="schema" v-model:selected="selected" />
          </div>
        </div>
      </div>
      <div class="footerBar">
        <button class="btn" @click="currentStepIndex = Math.max(0, currentStepIndex-1)">Back</button>
        <div style="display:flex; gap:8px">
          <button class="btn">Save as Template</button>
          <button class="btn primary" @click="currentStepIndex = Math.min(schema.steps.length-1, currentStepIndex+1)">Save & Next</button>
        </div>
      </div>
    </div>
  </div>

  <PreviewModal v-model:open="openPreview" :schema="schema" />
  <JsonPanel v-model:open="openJson" :schema="schema" />
  <div class="floating-eye" title="Open Preview" @click="openPreview = true">👁️</div>
</template>
