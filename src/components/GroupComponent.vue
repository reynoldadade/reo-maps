<script setup>
const emit = defineEmits(["changeColor", "changePointName"]);
const props = defineProps({
  coord: {
    type: Object,
    required: true,
  },
  index: {
    type: Number,
    required: false,
  },
  groupColors: {
    type: Array,
    required: true,
  },
});

function threeDp(value) {
  return value.toFixed(3);
}

function changeColor(index, length) {
  emit("changeColor", { index, length });
}

function changePointName(name) {
  emit("changePointName", name);
}
</script>

<template>
  <div class="mb-4">
    <button
      class="flex items-center gap-1 hover:bg-gray-200 p-1 rounded"
      @click="changeColor(index, coord.group.length)"
    >
      <div
        class="h-3 w-3 rounded-[2px]"
        :style="`background-color: ${groupColors[index]}`"
        v-if="coord.name !== 'Ungrouped'"
      ></div>
      <div class="flex h-full items-center pt-1">
        {{ coord.name !== "Ungrouped" ? "Group" : coord.name }}
      </div>
    </button>
    <div v-for="group in coord.group" :key="group.name" class="pl-1">
      <span
        role="button"
        class="hover:bg-gray-200 rounded"
        @click="changePointName(group.name)"
      >
        {{ group.name }}
      </span>
      <span class="text-[#767676] pl-2"
        >{{ threeDp(group.lng) }} {{ threeDp(group.lat) }}</span
      >
    </div>
  </div>
</template>
