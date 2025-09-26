<template>
    <div class="wrapper" @wheel="updateZoom" @click="toggleMove">
        <div class="screen" :style="screenStyle"></div>
    </div>
</template>


<script setup lang="ts">
import {computed, onBeforeMount, watch, reactive, ref} from "vue";
import {invoke} from "@tauri-apps/api/tauri";
import {window} from "@tauri-apps/api";
import {LogicalPosition, LogicalSize} from "@tauri-apps/api/window";

const props = defineProps<{
    screenshotPath: string;
    isActive: boolean;
}>();

const emit = defineEmits<{
    'update:screenshotPath': [path: string]
}>();

const WINDOW_SIZE_X = 256;
const WINDOW_SIZE_Y = 128;

const move = ref(true);
const lockOnScreen = ref(false);

const zoomLevel = ref(0);
const targetZoomLevel = ref(0);
const scale = computed(() => Math.pow(1.5, targetZoomLevel.value));

const cursor = {x: 0, y: 0};
const savedLocation = reactive({x: 0, y: 0});
const targetLocation = {x: 0, y: 0};
const targetContentCenter = {x: 0, y: 0};

// 鼠标静止检测
let mouseStillTimer: number | null = null;
let lastMouseX = 0;
let lastMouseY = 0;
let hasMovedSinceLastCapture = false;
const monitor = {
    size: {x: 0, y: 0},
    position: {x: 0, y: 0},
    scale: 1
}

let forceInstantMove = false;

const screenStyle = computed(() => {
    // 计算背景位置：让内容以鼠标为中心显示
    // savedLocation 现在是内容的中心点，我们需要调整它让内容在窗口中居中
    const centerOffsetX = WINDOW_SIZE_X / 2;
    const centerOffsetY = WINDOW_SIZE_Y / 2;

    return {
        backgroundImage: `url(${props.screenshotPath})`,
        backgroundPosition: `${-savedLocation.x + centerOffsetX + monitor.position.x / monitor.scale}px ${-savedLocation.y + centerOffsetY + monitor.position.y / monitor.scale}px`,
        backgroundSize: `${monitor.size.x / monitor.scale}px ${monitor.size.y / monitor.scale}px`,
    };
});

onBeforeMount(updateWindowSize);

watch(() => scale.value, updateWindowSize);

watch(() => props.isActive, async () => {
    if (props.isActive) {
        forceInstantMove = true;
        move.value = true;
        lockOnScreen.value = false;
        await moveLoop();
        await updateMonitor();
        lockOnScreen.value = true;
    } else {
        // 窗口隐藏时清除定时器，防止自动截图
        if (mouseStillTimer) {
            clearTimeout(mouseStillTimer);
            mouseStillTimer = null;
        }
        hasMovedSinceLastCapture = false;
    }
}, {
    immediate: true
});

async function updateMonitor() {
    const currentMonitor = await window.currentMonitor();
    if (!currentMonitor) {
        return;
    }
    const position = currentMonitor.position;
    const size = currentMonitor.size;
    const scaleFactor = currentMonitor.scaleFactor

    monitor.position = {x: position.x, y: position.y};
    monitor.size = {x: size.width, y: size.height};
    monitor.scale = scaleFactor;
}

async function updateZoom(event: WheelEvent) {
    zoomLevel.value -= event.deltaY / 100;
    zoomLevel.value = Math.max(0, zoomLevel.value);
    zoomLevel.value = Math.min(4, zoomLevel.value);
}

async function updateWindowSize() {
    await window.getCurrent().setSize(
        new LogicalSize(
            scale.value * WINDOW_SIZE_X,
            scale.value * WINDOW_SIZE_Y
        )
    );
}

async function windowMove() {
    const mouseLocation = await invoke("get_mouse_location", {});
    const location = mouseLocation.split(";").map(v => parseInt(v) / monitor.scale);
    cursor.x = location[0];
    cursor.y = location[1];

    // 窗口位置：鼠标右下方保持距离
    const OFFSET_X = 20; // 向右偏移20像素
    const OFFSET_Y = 20; // 向下偏移20像素
    targetLocation.x = cursor.x + OFFSET_X;
    targetLocation.y = cursor.y + OFFSET_Y;

    // 内容显示：以鼠标为中心
    targetContentCenter.x = cursor.x;
    targetContentCenter.y = cursor.y;

    // 检测鼠标是否移动，如果移动则重置定时器
    if (cursor.x !== lastMouseX || cursor.y !== lastMouseY) {
        lastMouseX = cursor.x;
        lastMouseY = cursor.y;
        hasMovedSinceLastCapture = true; // 标记鼠标已移动

        if (mouseStillTimer) {
            clearTimeout(mouseStillTimer);
        }

        // 鼠标移动后静止2秒后刷新截图
        mouseStillTimer = setTimeout(async () => {
            if (!hasMovedSinceLastCapture) return; // 如果没有移动过，不执行截图
            // 先隐藏窗口，再截图，再显示
            const { window } = await import("@tauri-apps/api");
            await window.getCurrent().hide();

            await new Promise(resolve => setTimeout(resolve, 100)); // 等待100ms确保窗口隐藏

            const { convertFileSrc, invoke } = await import("@tauri-apps/api/tauri");
            const newScreenshotPath = convertFileSrc((await invoke("update_screenshot", {})).replaceAll("\\", "/")) + "?" + Date.now();

            await window.getCurrent().show();
            emit('update:screenshotPath', newScreenshotPath);
            hasMovedSinceLastCapture = false; // 重置移动标记
        }, 2000);
    }

    const moveWindow = async () => {
        await window.appWindow.setPosition(getWindowPosition());
    };

    await Promise.all([
        updateSavedLocation(),
        moveWindow()
    ]);
}

function getWindowPosition() {
    if (!lockOnScreen.value) {
        return new LogicalPosition(
            targetLocation.x,
            targetLocation.y
        );
    }

    return new LogicalPosition(
        clamp(targetLocation.x, monitor.position.x, monitor.position.x + monitor.size.x / monitor.scale - scale.value * WINDOW_SIZE_X),
        clamp(targetLocation.y, monitor.position.y, monitor.position.y + monitor.size.y / monitor.scale - scale.value * WINDOW_SIZE_Y)
    );

    function clamp(value: number, min: number, max: number) {
        return Math.min(Math.max(value, min), max);
    }
}

async function updateSavedLocation() {
    const alpha = forceInstantMove ? 1 : 0.3;
    forceInstantMove = false;

    targetZoomLevel.value = lerp(targetZoomLevel.value, zoomLevel.value, alpha);

    savedLocation.x = lerp(savedLocation.x, targetContentCenter.x, alpha);
    savedLocation.y = lerp(savedLocation.y, targetContentCenter.y, alpha);

    function lerp(a: number, b: number, alpha: number) {
        return a * (1 - alpha) + b * alpha;
    }
}

async function moveLoop() {
    await windowMove();
    if (props.isActive && move.value) {
        requestAnimationFrame(moveLoop);
    }
}

async function toggleMove() {
    move.value = !move.value;
    await moveLoop();
}
</script>

<style scoped lang="scss">
.wrapper {
    border-radius: 16px;
    width: 100%;
    height: 100%;
    overflow: hidden;
    display: flex;
    position: relative;

    &::after {
        content: "";
        position: absolute;
        inset: 0;
        border-radius: 16px;
        box-shadow: inset 0 0 0 1px rgba(0, 0, 0, 0.2);
    }

    .screen {
        image-rendering: pixelated;
        opacity: 1;
        width: 100%;
        height: 100%;
        transform: scale(v-bind(scale));
        background-repeat: no-repeat;
    }
}
</style>
