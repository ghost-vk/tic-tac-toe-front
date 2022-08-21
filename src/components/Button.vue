<script setup>
const props = defineProps({ text: String });
</script>

<template>
  <button
    class="px-3 py-2 bg-green-600 hover:bg-green-500 text-white shadow rounded uppercase"
  >
    {{ props.text }}
  </button>
</template>
