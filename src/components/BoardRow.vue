<script setup>
import BoardCell from './BoardCell.vue'
defineProps({ cells: Array })

const emit = defineEmits(['cellClick'])

const onCellClick = (idx) => {
  emit('cellClick', { x: idx })
}
</script>

<template>
  <div class="flex flex-nowrap w-full">
    <BoardCell v-for="(cell, idx) in cells" :cell="cell" @cellClick="onCellClick(idx)" />
  </div>
</template>
