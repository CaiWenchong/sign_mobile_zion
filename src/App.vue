<script setup lang="ts">
  import Downloader from "./components/Downloader.vue";
  import { reactive, ref } from "vue";
  
  const globalData = reactive({ value: "http://shipin.cdxyhpx.com/sv/3bba3a21-18926843f74/3bba3a21-18926843f74.mp4" });
  const setGlobalData = (obj: any) => {
    globalData.value = obj.value;
  }
  
  
  
  </script>
  
  <template>
    <Downloader source="http://shipin.cdxyhpx.com/sv/3bba3a21-18926843f74/3bba3a21-18926843f74.mp4"
      vid="a175d6b01b4271ee80980764b3ec0102" playauth="" :global-data="globalData"
      :set-global-data="setGlobalData" :course_log_pk="0" />
  </template>
  
  <style>
  #app{
    height: 60vh;
  }
  </style>
  