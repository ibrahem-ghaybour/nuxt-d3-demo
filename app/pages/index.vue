<!-- pages/index.vue (كما هي تقريبًا) -->
<script setup lang="ts">
import { ref } from "vue";
import ThreeBars from "~/components/ThreeBars.vue";

type Row = { label: string; value: number };

const data = ref<Row[]>([
  { label: "Vue", value: 38 },
  { label: "Nuxt", value: 45 },
  { label: "D3", value: 28 },
  { label: "Three.js", value: 52 },
  { label: "Tailwind", value: 41 },
]);

function randomize() {
  data.value = data.value.map((d) => ({
    ...d,
    value: Math.max(5, Math.round(d.value * (0.7 + Math.random() * 0.6))),
  }));
}

function addBar() {
  const next = data.value.length + 1;
  data.value = [
    ...data.value,
    { label: `Item ${next}`, value: Math.round(10 + Math.random() * 60) },
  ];
}
</script>

<template>
  <main class="min-h-screen bg-slate-950 text-slate-100">
    <section class="max-w-5xl mx-auto p-6">
      <h1 class="text-2xl font-semibold mb-2">
        Nuxt 3 + Three.js — 3D Bars (Polished)
      </h1>
      <p class="text-slate-300 mb-4">
        إضاءة واقعية، ظلال ناعمة، تدرّج ألوان، وتحريك سلس.
      </p>

      <div class="flex gap-2 mb-3">
        <button
          class="px-3 py-1.5 rounded-lg bg-slate-800 hover:bg-slate-700"
          @click="randomize"
        >
          Randomize
        </button>
        <button
          class="px-3 py-1.5 rounded-lg bg-indigo-600 hover:bg-indigo-500"
          @click="addBar"
        >
          Add Bar
        </button>
      </div>

      <ThreeBars
        :data="data"
        :height="540"
        :background="'#0b1220'"
        :autoRotate="true"
      />
    </section>
  </main>
</template>
