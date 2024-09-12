<script setup>
import InputNumber from 'primevue/inputnumber'
import InputGroup from 'primevue/inputgroup'
import InputGroupAddon from 'primevue/inputgroupaddon'
import Button from 'primevue/button'
import { ref, onMounted, computed } from 'vue'
import { jsPDF } from 'jspdf'
import PDF417 from 'pdf417-generator'
import scissors from './assets/artboard.png'
const SKU = ref('YOUR SKU HERE')
const copies = ref(4)
function onRecordChange(record) {
  const mapped = grist.mapColumnNames(record)
  SKU.value = mapped.SKU
}

onMounted(() => {
  grist.ready({
    columns: ['SKU'],
    requiredAccess: 'read table'
  })

  grist.onRecord(onRecordChange)
})

function autoScaleText(doc, text, startX, startY, maxWidth, maxHeight) {
  doc.setFontSize(1)
  const size = doc.getTextDimensions(text)
  const { w, h } = size
  const tW = maxWidth / w
  const tH = maxHeight / h
  doc.setFontSize(Math.min(tW, tH))
  doc.text(text, startX, startY, { align: 'center' })
}

const totalHeight = computed(() => {
  return 0.8 * copies.value + 0.4 * (copies.value == 1 ? 0 : 1)
})

const label = computed(() => {
  const barcode = makePDF417DataUri(SKU.value)
  const height = 0.8
  const width = 2.4
  const leader = 0.4
  const totalHeight = height * copies.value + leader * (copies.value == 1 ? 0 : 1)
  const options = {
    orientation: width > totalHeight ? 'l' : 'p',
    unit: 'in',
    format: [width, totalHeight]
  }
  const doc = new jsPDF(options)
  doc.setLineDashPattern([0.1, 0.05, 0.2, 0.05], 0)
  doc.setLineWidth(0.01)
  doc.setFont('Helvetica')

  function buildLabel(yOffset) {
    doc.addImage(barcode, 'PNG', 0, yOffset + 0.1, width, 0.4)
    autoScaleText(doc, SKU.value, 1.2, yOffset + 0.6, width, 0.1)
  }

  function drawCutLine(yOffset) {
    doc.setDrawColor(50, 50, 50)
    doc.addImage(scissors, 'PNG', 0.1, yOffset - 0.15, 0.15, 0.15)
    doc.line(0, yOffset, width, yOffset)
  }
  if (copies.value > 1) {
    autoScaleText(doc, SKU.value, 1.2, 0.2, 2.4, 0.1)
    autoScaleText(doc, `Copies: ${copies.value}`, 1.2, 0.3, 2.4, 0.1)
    drawCutLine(leader)
  }
  for (let i = 0; i < copies.value; i++) {
    const labelStart = i * height + leader * (copies.value == 1 ? 0 : 1)
    buildLabel(labelStart)
    if (i < copies.value - 1) {
      drawCutLine(labelStart + height)
    }
  }
  const data = doc.output('datauristring')
  return data
})

function makePDF417DataUri(value) {
  const canvas = document.createElement('canvas')
  PDF417.draw(value, canvas)
  return canvas.toDataURL()
}
</script>

<template>
  <div class="container bg-blue-100 h-full">
    <div class="flex flex-row gap-6 h-full">
      <embed :src="label" height="100%" />
      <div class="flex flex-col gap-2 text-black">
        <InputGroup>
          <InputGroupAddon>Copies</InputGroupAddon>
          <InputNumber v-model="copies" showButtons buttonLayout="horizontal" :min="1" />
          <Button label="Reset" @click="copies = 1">Reset</Button>
        </InputGroup>
        Total Length: {{ totalHeight }} in
      </div>
    </div>
  </div>
</template>

<style scoped></style>
