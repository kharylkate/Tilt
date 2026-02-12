<template>
  <pre class="output">
  {{ outputText }}
  </pre>
  <button class="btn-main" :class="[isButtonDisabled && 'clicked']" @click="startGame">
    {{ !isButtonDisabled ? "Start" : "End" }} (Will
    {{ isButtonDisabled ? "enable" : "disable" }} buttons)
  </button>

  <div class="viewport">
    <div
      class="world"
      :style="{
        transform: `translate(${-camera.x}px, ${-camera.y}px)`
      }"
    >
      <div
        v-for="dot in dots"
        :key="dot.id"
        class="dot"
        :style="{
          left: dot.x + 'px',
          top: dot.y + 'px',
          backgroundColor: dot.color
        }"
      />

        <div class="circle" ref="circleRef"></div>
      <!-- <div
        class="circle"
        :style="{
          left: circle.x + 'px',
          top: circle.y + 'px'
        }"
      /> -->
    </div>
  </div>

  <button @click="start">Start</button>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, watch, reactive } from "vue";
import { SENSITIVITY, LEVEL, POSITION } from "../lib/constants";

/* props (optional, unchanged) */
defineProps<{
  msg: string;
}>();

// =========================
// STATE
// =========================
const screenRef = ref<HTMLDivElement | null>(null);
const circleRef = ref<HTMLDivElement | null>(null);
const tiltX = ref<number>(0);
const tiltY = ref<number>(0);
const outputText = ref<string>("");
const isButtonDisabled = ref<boolean>(false);
const difficulty = ref<string>("easy"); // could be "easy", "medium", "hard" - affects sensitivity
const sensitivity = ref<number>(SENSITIVITY.LOW); // multiplier for tilt sensitivity, can be adjusted based on difficulty
const level = ref<number>(LEVEL.LEVEL1); // game level, can increase over time to make it harder

const circleXPosition = ref<number>(0);
const circleYPosition = ref<number>(0);




/* Constants */
const WORLD_SIZE = 1500
const VIEWPORT_SIZE = 400
const CIRCLE_SIZE = 50
const RADIUS = CIRCLE_SIZE / 2

/* State */
const circle = reactive({ x: 0, y: 0 })
const camera = reactive({ x: 0, y: 0 })
const isAnimating = ref(false)

/* Decorations */
const dots = ref(
  Array.from({ length: 250 }, (_, i) => ({
    id: i,
    x: Math.random() * (WORLD_SIZE - 20),
    y: Math.random() * (WORLD_SIZE - 20),
    color: `hsl(${Math.random() * 360}, 70%, 50%)`
  }))
)

/* 🚨 WATCH: camera follows circle */
watch(
  () => ({ x: circle.x, y: circle.y }),
  ({ x, y }) => {
    let camX = x - VIEWPORT_SIZE / 2 + RADIUS
    let camY = y - VIEWPORT_SIZE / 2 + RADIUS

    camera.x = Math.max(0, Math.min(camX, WORLD_SIZE - VIEWPORT_SIZE))
    camera.y = Math.max(0, Math.min(camY, WORLD_SIZE - VIEWPORT_SIZE))
  }
)

/* Animation loop (time driver) */
function start() {
  if (isAnimating.value) return
  isAnimating.value = true

  circle.x = 0
  circle.y = 0

  const speedX = 2
  const speedY = 1.5
  const targetX = 900
  const targetY = 900

  function animate() {
    if (!isAnimating.value) return

    if (circle.x < targetX) circle.x += speedX
    if (circle.y < targetY) circle.y += speedY

    if (circle.x < targetX || circle.y < targetY) {
      requestAnimationFrame(animate)
    } else {
      isAnimating.value = false
    }
  }

  requestAnimationFrame(animate)
}

// =========================
// SCREEN ORIENTATION
// =========================
const getScreenOrientation = (): string => {
  return (
    screen.orientation?.type ||
    (window.innerWidth > window.innerHeight ? "landscape" : "portrait")
  );
};

const renderScreenOrientation = (): string => {
  return `Screen orientation: ${getScreenOrientation()}`;
};

// =========================
// DEVICE TILT
// =========================
const onDeviceMotion = (event: DeviceMotionEvent): void => {
  const a = event.accelerationIncludingGravity;
  if (!a) return;

  // smoothing
  tiltX.value = tiltX.value * 0.8 + (a.x ?? 0) * 0.2;
  tiltY.value = tiltY.value * 0.8 + (a.y ?? 0) * 0.2;

  renderTiltScreen();
};

const startDeviceTilt = (): void => {
  window.addEventListener("devicemotion", onDeviceMotion);
};

// =========================
// RENDER
// =========================
const renderTiltScreen = (): void => {
  let horizontal: string;
  if (tiltX.value < -2) horizontal = "Tilting LEFT";
  else if (tiltX.value > 2) horizontal = "Tilting RIGHT";
  else horizontal = "Level horizontally";

  let vertical: string;
  if (tiltY.value < -2) vertical = "Tilting FORWARD";
  else if (tiltY.value > 2) vertical = "Tilting BACKWARD";
  else vertical = "Level vertically";

  outputText.value = `
${renderScreenOrientation()}

Horizontal tilt: ${horizontal} (x=${tiltX.value.toFixed(2)})
Vertical tilt  : ${vertical} (y=${tiltY.value.toFixed(2)})
  `.trim();
};

// =========================
// ORIENTATION CHANGE
// =========================
const onOrientationChange = (): void => {
  renderTiltScreen();
};

// =========================
// MOVE OBJECT BY CLICK
// =========================
const moveCircle = (axis: string, movement: string): void => {
  const circle = document.querySelector(".circle") as HTMLElement;
  if (!circle) return;

  if (axis === "x") {
    circleXPosition.value += movement === "+" ? 5 : -5;
  }

  if (axis === "y") {
    circleYPosition.value += movement === "+" ? -5 : 5;
  }

  circle.style.left = `${circleXPosition.value}px`;
  circle.style.top = `${circleYPosition.value}px`;
};

// ~~~~~~~~~~~~~~~~~~~~~~~~~
// SETTERS

// =========================
// SET OBJECT SENSITIVITY
// =========================
const setSensitivity = (): void => {
  console.log("Current sensitivity:", sensitivity.value);
  if (sensitivity.value === SENSITIVITY.LOW) {
    sensitivity.value = SENSITIVITY.MEDIUM;
    return;
  }
  if (sensitivity.value === SENSITIVITY.MEDIUM) {
    sensitivity.value = SENSITIVITY.HIGH;
    return;
  }
  if (sensitivity.value === SENSITIVITY.HIGH) {
    sensitivity.value = SENSITIVITY.LOW;
    return;
  }
};

// =========================
// SET OBJECT SENSITIVITY
// =========================
const setObjectPositionByLevel = (): void => {
  switch (level.value) {
    case LEVEL.LEVEL1:
      circleXPosition.value = POSITION.LEVEL1.x;
      circleYPosition.value = POSITION.LEVEL1.y;
      break;
    case LEVEL.LEVEL2:
      circleXPosition.value = POSITION.LEVEL2.x;
      circleYPosition.value = POSITION.LEVEL2.y;
      break;
    case LEVEL.LEVEL3:
      circleXPosition.value = POSITION.LEVEL3.x;
      circleYPosition.value = POSITION.LEVEL3.x;
      break;
    default:
      circleXPosition.value = POSITION.LEVEL1.x;
      circleYPosition.value = POSITION.LEVEL1.y;
  }

  if (circleRef.value) {
    circleRef.value.style.left = `${circleXPosition.value}px`;
    circleRef.value.style.top = `${circleYPosition.value - 107}px`;
  }
};

const startGame = (): void => {
  isButtonDisabled.value = !isButtonDisabled.value;

  if (isButtonDisabled.value) {
  }
  // reset tilt
  // tiltX.value = 0;
  // tiltY.value = 0;

  startDeviceTilt();
};

// ~~~~~~~~~~~~~~~~~~~~~~~~~
// WATCHERS

// =========================
// MOVE OBJECT BY TILT
// =========================
watch([tiltX, tiltY], ([newTiltX, newTiltY]) => {
  const circle = document.querySelector(".circle") as HTMLElement;
  if (!circle || !isButtonDisabled.value) return;

  const tolerance = 0.1;
  const initialTiltX = 0;
  const initialTiltY = 0;

  if (!circleRef.value) return;

  circleXPosition.value += -newTiltX;
  circleYPosition.value += newTiltY;
  const scaledX = circleXPosition.value * sensitivity.value;
  const scaledY = circleYPosition.value * sensitivity.value;

  circleRef.value.style.left = `${scaledX}px`;
  circleRef.value.style.top = `${scaledY}px`;
});

// =========================
// LIFECYCLE
// =========================
onMounted(() => {
  setObjectPositionByLevel();

  window.addEventListener("orientationchange", onOrientationChange);

  if (screen.orientation) {
    screen.orientation.addEventListener("change", onOrientationChange);
  }

  startDeviceTilt();
});

onBeforeUnmount(() => {
  window.removeEventListener("orientationchange", onOrientationChange);
  window.removeEventListener("devicemotion", onDeviceMotion);

  if (screen.orientation) {
    screen.orientation.removeEventListener("change", onOrientationChange);
  }
});
</script>

<style scoped>
.screen {
  position: relative;
  width: 100%;
  height: 300px;
  background-color: #f0f0f0;
  border: 1px solid #ccc;
  margin-top: 20px;

  .circle {
    width: 70px;
    height: 70px;
    background-color: blue;
    border-radius: 50%;
    position: absolute;
  }
}

.btn-main {
  display: block;
  margin: 20px auto;
  padding: 10px 20px;
  font-size: 1rem;
  cursor: pointer;
  background-color: #28a745;
  color: white;
  border: none;
  border-radius: 5px;

  &.clicked {
    background-color: rgb(253, 72, 72) !important;
  }

  &:hover {
    background-color: #218838;
  }
}
.btn-container {
  display: flex;
  justify-content: center;
  width: 100%;
  margin-top: 20px;

  .btn-movement {
    padding: 10px 20px;
    margin: 0 10px;
    font-size: 1rem;
    cursor: pointer;
    background-color: #007bff;
    color: white;
    border: none;
    border-radius: 5px;

    &:hover {
      background-color: #0056b3;
    }

    &.disabled {
      background-color: #6c757d;
      cursor: not-allowed;
    }
  }
}

h1 {
  font-weight: 500;
  font-size: 2.6rem;
  position: relative;
  top: -10px;
}

h3 {
  font-size: 1.2rem;
}

.greetings h1,
.greetings h3 {
  text-align: center;
}

@media (min-width: 1024px) {
  .greetings h1,
  .greetings h3 {
    text-align: left;
  }
}

/* backgrounds */
.background1 {
  position: absolute;
  display: flex;
  align-items: center;
  justify-items: center;
  top: 8px;
  left: 0;
  width: 100%;
  height: 95%;
  /* background: pink; */

  .straight-road {
    position: relative;

    /* top: 0; */

    width: 100%;
    height: 60%;
    /* height: 100%; */
    background-color: gray;
  }
}



.viewport {
  width: 400px;
  height: 400px;
  overflow: hidden;
  border: 2px solid black;
  position: relative;
  background: #e0f7ff;
}

.world {
  position: absolute;
  width: 1500px;
  height: 1500px;
  background-color: #ccc;
}

.circle {
  width: 50px;
  height: 50px;
  background: red;
  border-radius: 50%;
  position: absolute;
  z-index: 10;
  box-shadow: 0 0 8px rgba(0, 0, 0, 0.5);
}

.dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  position: absolute;
}
</style>
