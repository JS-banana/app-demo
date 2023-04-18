<template>
  <div>
    <button @click="handleClick">添加</button>
    <ul ref="listContainerRef"></ul>
  </div>
</template>
<script setup lang="ts">
import { ref } from "vue";

const listContainerRef = ref<HTMLUListElement>();

const handleClick = () => {
  let now = Date.now();
  const total = 10000;

  for (let i = 0; i < total; i++) {
    const li = document.createElement("li");
    li.innerText = `a-${i + 1}`;
    const container = listContainerRef.value;
    container?.appendChild(li);
  }

  console.log("js运行时间", Date.now() - now);

  setTimeout(() => {
    console.log("总运行时间", Date.now() - now);
  }, 0);
};
</script>
