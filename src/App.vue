<script setup lang="ts">
import Sign from "./components/Sign.vue";
import { reactive, ref } from "vue";

const globalData = reactive({ componentName: "SignName" });
const setGlobalData = (obj: any) => {
  globalData.value = obj.value;
}

// const url = ref("https://zion-app.functorz.com/zero/JmAxbl1kYqo/api/graphql-v2");
// const actionflow_id = ref("5e5e155d-7b3c-4771-bf94-ceb597cbb666")

const url = ref("");
const actionflow_id = ref("")

</script>

<template>
  <Sign 
      :global-data="globalData"
    :set-global-data="setGlobalData" :course_log_pk="0" />
</template>

<style>
#app{
  height: 60vh;
}
</style>
