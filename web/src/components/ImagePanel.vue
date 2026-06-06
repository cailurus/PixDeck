<script setup lang="ts">
import { ref, onUnmounted } from 'vue'
import { useCanvas, PRESET } from '../canvas/useCanvas'
import { pixelize, type PixelizeMethod } from '../canvas/pixelize'

const c = useCanvas()
const dataUrl = ref<string>('')
const size = ref(13)
const method = ref<PixelizeMethod>('mode')
const snap = ref(true)
const invert = ref(false)

function revokeBlob() {
  if (dataUrl.value.startsWith('blob:')) URL.revokeObjectURL(dataUrl.value)
}
onUnmounted(revokeBlob)
function onFile(e: Event) {
  const f = (e.target as HTMLInputElement).files?.[0]
  if (!f) return
  revokeBlob() // 释放上一张上传图
  dataUrl.value = URL.createObjectURL(f)
}
function loadImage(src: string): Promise<HTMLImageElement> {
  return new Promise((resolve, reject) => {
    const img = new Image()
    img.onload = () => resolve(img)
    img.onerror = () => reject(new Error('image load failed'))
    img.src = src
  })
}
async function generate() {
  if (!dataUrl.value) { c.status.value = '先上传一张图片'; return }
  let img: HTMLImageElement
  try { img = await loadImage(dataUrl.value) } catch { c.status.value = '图片加载失败'; return }
  const off = document.createElement('canvas')
  off.width = img.width; off.height = img.height
  const ictx = off.getContext('2d')!
  ictx.drawImage(img, 0, 0)
  const id = ictx.getImageData(0, 0, off.width, off.height)
  const block = pixelize({ width: id.width, height: id.height, data: id.data }, {
    size: size.value, method: method.value, snap: snap.value, invert: invert.value, palette: PRESET,
  })
  if (!block) { c.status.value = '没识别到主体,换张图或调一下反相试试'; return }
  c.placeBlock(block.cells, 0, 0)
}
</script>

<template>
  <div class="imgpanel">
    <div class="col">
      <div class="lbl">来源</div>
      <label class="upload">上传图片 <input type="file" accept="image/*" @change="onFile" /></label>
      <div class="hint">选一张图(Logo / 图标 / 任意图片),下面像素化后落到画布,可再用画笔、文字调整。</div>
    </div>
    <div class="col">
      <div class="lbl">像素化</div>
      <label class="fld">大小 <input type="range" min="8" max="16" v-model.number="size" /><span class="v">{{ size }}</span></label>
      <label class="fld">方法
        <select v-model="method"><option value="mode">主色投票</option><option value="nearest">最近邻</option><option value="smooth">平滑</option></select>
      </label>
      <label class="chk"><input type="checkbox" v-model="snap" /> 吸附纯色</label>
      <label class="chk"><input type="checkbox" v-model="invert" /> 暗色当主体(纯黑图标)</label>
      <button class="gen" @click="generate">↓ 生成到画布</button>
    </div>
  </div>
</template>

<style scoped>
.imgpanel{display:flex;gap:20px;flex-wrap:wrap;background:var(--bg-1);border:1px solid var(--line);border-radius:12px;padding:14px}
.col{display:flex;flex-direction:column;gap:8px;min-width:200px}
.lbl{font-family:var(--mono);font-size:var(--fs-micro);letter-spacing:.18em;text-transform:uppercase;color:var(--ink-3)}
.upload{font-size:var(--fs-body);color:var(--ink-2)}
.hint{font-size:var(--fs-small);color:var(--ink-3);max-width:260px;line-height:1.55}
.fld{display:flex;align-items:center;gap:8px;color:var(--ink-2);font-size:var(--fs-body)}
.fld .v{font-family:var(--mono);color:var(--ink)}
.chk{display:flex;align-items:center;gap:6px;color:var(--ink-2);font-size:var(--fs-body)}
select{background:var(--bg-2);color:var(--ink);border:1px solid var(--line-2);border-radius:7px;padding:6px 8px;font-family:var(--mono);font-size:var(--fs-body)}
.gen{margin-top:4px;cursor:pointer;border:1px solid var(--signal-dim);background:var(--bg-2);color:var(--signal);padding:7px 12px;border-radius:8px;font-weight:600}
.gen:hover{background:var(--bg-3)}
</style>
