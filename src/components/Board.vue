<script setup>
import BoardRow from './BoardRow.vue';
defineProps({ rows: Array });

const emit = defineEmits(['cellClick']);

const onCellClick = (idx, xCoord) => {
  emit('cellClick', { y: idx, ...xCoord });
}

</script>

<template>
  <div class="flex h-full justify-center items-center">
    <div>
      <BoardRow v-for="(row, idx) in rows" :cells="row" @cellClick="(e) => onCellClick(idx, e)" />
    </div>
  </div>
</template>
