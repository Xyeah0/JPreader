<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue"
import JpReader from "./components/JpReader.vue"
import MobileReader from "./components/MobileReader.vue"

const isMobile = ref(window.innerWidth <= 600)
function checkMobile() {
  isMobile.value = window.innerWidth <= 600
}
onMounted(() => {
  window.addEventListener("resize", checkMobile)
  checkMobile()
})
onBeforeUnmount(() => {
  window.removeEventListener("resize", checkMobile)
})
</script>

<template>
  <MobileReader v-if="isMobile" />
  <JpReader v-else />
</template>
