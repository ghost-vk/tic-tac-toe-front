<script setup>
const props = defineProps({
  cell: String || null
});

const emit = defineEmits(['cellClick']);

const onClick = () => {
  if (props.cell !== null) return;
  emit('cellClick');
}
</script>

<template>
  <div class="w-10 h-10 p-1 cursor-pointer">
    <div
      class="flex w-9 h-9 rounded bg-green-200 hover:bg-green-300 items-center justify-center text-lg"
      @click="onClick"
    >
      {{ cell }}
    </div>
  </div>
</template>
