<template>
  <div ref="refContainer" class="container" @scroll="handleScroll">
    <div
      class="infinite-list-height"
      :style="{ height: `${listHeight}px` }"
    ></div>
    <div class="infinite-list" :style="{ transform: transFormOffset }">
      <div
        class="infinite-list-item"
        v-for="item in visibleData"
        :key="item.id"
        :style="{ height: `${itemSize}px`, lineHeight: `${itemSize}px` }"
      >
        {{ item.value }}
      </div>
    </div>
  </div>
</template>
<script setup lang="ts">
import { computed, onMounted, reactive, ref } from "vue";

const refContainer = ref();

const props = defineProps<{
  listData: Array<any>;
  itemSize: number;
}>();

const data = reactive({
  // 可视区域高度
  screenHeight: 0,
  // 偏移量
  startOffset: 0,
  // 起始索引
  start: 0,
  // 结束索引
  end: 0,
});

// 列表总高度
const listHeight = computed(() => props.listData.length * props.itemSize);
// 可显示的列表项
const visibleCount = computed(() => data.screenHeight / props.itemSize);
// 偏移量
const transFormOffset = computed(
  () => `translate3d(0,${data.startOffset}px,0)`
);
// 实际显示的数据
const visibleData = computed(() =>
  props.listData.slice(data.start, Math.min(data.end, props.listData.length))
);

onMounted(() => {
  data.screenHeight = refContainer.value.clientHeight;
  data.end = data.start + visibleCount.value;
});

const handleScroll = () => {
  // 当前偏移量
  const scrollTop = refContainer.value.scrollTop;
  console.log("scrollTop", scrollTop);

  // 开始索引
  data.start = Math.floor(scrollTop / props.itemSize);
  //结束索引
  data.end = data.start + visibleCount.value;
  // 此时的偏移量
  data.startOffset = scrollTop - (scrollTop % props.itemSize);

  console.log("data.startOffset", data.startOffset);
};
</script>
<style scoped>
.container {
  width: 360px;
  height: 500px;
  position: relative;
  overflow: auto;
  background-color: #f3f4f7;
}
.infinite-list-height {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  z-index: -1;
}
.infinite-list {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
}
.infinite-list-item {
  padding: 4px 8px;
  color: coral;
  border-bottom: 1px solid #d7dadf;
}
</style>
