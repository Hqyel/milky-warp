<template>
    <MZoom
        :screenshot-path="screenshotPath"
        :is-active="isWindowDisplayed"
        :zoom-level="zoomLevel"
        :class="{hide: holdHide}"
        @update:screenshotPath="screenshotPath = $event"
    />
</template>

<script setup lang="ts">
import MZoom from "./components/Zoom.vue";
import { register } from "@tauri-apps/api/globalShortcut";
import { window } from "@tauri-apps/api";
import { convertFileSrc, invoke } from "@tauri-apps/api/tauri";
import { nextTick, onMounted, ref, watch } from "vue";
import { config } from "./config";
import { useMagicKeys } from "@vueuse/core";

const keys = useMagicKeys();
const holdShortcut = keys[config.holdShortcut];

const isWindowDisplayed = ref(false);
const screenshotPath = ref("");
const zoomLevel = ref(0);

const holdHide = ref(false);

onMounted(async () => {
    screenshotPath.value = convertFileSrc((await invoke("get_screenshot_path", {})).replaceAll("\\", "/"));
});

register(config.shortcut, async () => {
    isWindowDisplayed.value = !isWindowDisplayed.value;

    await nextTick(); // Needed to avoid position flickering

    if (!isWindowDisplayed.value) {
        await window.getCurrent().hide();
    }
    else {
        holdHide.value = false;
        screenshotPath.value = convertFileSrc((await invoke("update_screenshot", {})).replaceAll("\\", "/")) + "?" + Date.now();
        await window.getCurrent().show();
        //await window.getCurrent().setFocus();
    }
});

// 注册缩放快捷键
register(config.zoomInShortcut, async () => {
    console.log('Zoom in shortcut triggered, isActive:', isWindowDisplayed.value, 'current zoom:', zoomLevel.value);
    if (isWindowDisplayed.value) {
        zoomLevel.value = Math.min(zoomLevel.value + 1, 10); // 最大缩放10级
        console.log('Zoom in: new zoom level:', zoomLevel.value);
    }
});

register(config.zoomOutShortcut, async () => {
    console.log('Zoom out shortcut triggered, isActive:', isWindowDisplayed.value, 'current zoom:', zoomLevel.value);
    if (isWindowDisplayed.value) {
        zoomLevel.value = Math.max(zoomLevel.value - 1, -5); // 最小缩放-5级
        console.log('Zoom out: new zoom level:', zoomLevel.value);
    }
});

watch(holdShortcut, async (value) => {
    if (value) {
        if (isWindowDisplayed.value) {
            return
        }

        isWindowDisplayed.value = true;
        await nextTick(); // Needed to avoid position flickering
        screenshotPath.value = convertFileSrc((await invoke("update_screenshot", {})).replaceAll("\\", "/")) + "?" + Date.now();
        holdHide.value = false;
    }
    else {
        holdHide.value = true;
        isWindowDisplayed.value = false;
    }
});

</script>

<style lang="scss" scoped>
.hide {
    display: none;
}
</style>

<style lang="scss">
html, body {
    margin: 0;
    background-color: transparent;

    #app {
        height: 100vh;
        overflow-y: auto;
    }
}
</style>
