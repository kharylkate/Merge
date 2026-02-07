<template>
  <pre class="output">
  {{ outputText }}
  </pre>
  <button class="btn-main" :class="[isButtonDisabled && 'clicked']" @click="startGame">
    {{ !isButtonDisabled ? "Start" : "End" }} (Will
    {{ isButtonDisabled ? "enable" : "disable" }} buttons)
  </button>
  <div class="btn-container">
    <button
      class="btn-movement"
      :class="[isButtonDisabled && 'disabled']"
      :disabled="isButtonDisabled"
      @click="moveCircle('x', '-')"
    >
      Move left
    </button>
    <button
      class="btn-movement"
      :class="[isButtonDisabled && 'disabled']"
      @click="moveCircle('x', '+')"
    >
      Move right
    </button>
    <button
      class="btn-movement"
      :class="[isButtonDisabled && 'disabled']"
      @click="moveCircle('y', '+')"
    >
      Move up
    </button>
    <button
      class="btn-movement"
      :class="[isButtonDisabled && 'disabled']"
      @click="moveCircle('y', '-')"
    >
      Move down
    </button>
  </div>
  <h3>X position: {{ circleXPosition }}</h3>
  <h3>Y position: {{ circleYPosition }}</h3>
  <div class="screen" ref="screenRef">
    <div class="background1" ref="backgroundLevel1">
      <div class="straight-road"></div>
    </div>
    <div class="circle" ref="circleRef"></div>
  </div>

  <div class="btn-container">
    <button class="btn-movement" @click="setSensitivity">
      Sensitivity: {{ sensitivity }}
    </button>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, watch } from "vue";
import { SENSITIVITY, LEVEL, POSITION } from "../lib/constants";

/* props (optional, unchanged) */
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

  const tolerateX = Math.abs(newTiltX - initialTiltX) <= tolerance;
  const tolerateY = Math.abs(newTiltY - initialTiltY) <= tolerance;

  if (tolerateX && tolerateY) {
    // within tolerance, do not move
    if (circleRef.value) circleRef.value.style.backgroundColor = "yellow";
    return;
  }

  // ===============================
  // DETECT BEYOND SCREEN MOVEMENT
  // ===============================

  if (!screenRef.value || !circleRef.value) return;

  const screenRect = screenRef.value.getBoundingClientRect();
  const circleRect = circleRef.value.getBoundingClientRect();

  // Check if circle is outside the container
  const isOutside =
    circleRect.left + 5 < screenRect.left ||
    circleRect.right + 5 > screenRect.right ||
    circleRect.top < screenRect.top ||
    circleRect.bottom + 5 > screenRect.bottom;

  if (isOutside) {
    console.warn("Circle is moving beyond the screen boundaries!");
    circleRef.value.style.backgroundColor = "orange";
    screenRef.value.style.borderColor = "red";
    screenRef.value.style.borderWidth = "5px";
    screenRef.value.style.borderStyle = "solid";
    return;
  } else {
    circleRef.value.style.backgroundColor = "blue"; // or your default
  }

  circleXPosition.value += -newTiltX;
  circleYPosition.value += newTiltY;
  // circle.style.left = `${ circleXPosition.value * sensitivity.value }px`;
  // circle.style.top = `${ circleYPosition.value * sensitivity.value }px`;
  const scaledX = circleXPosition.value * sensitivity.value;
  const scaledY = circleYPosition.value * sensitivity.value;

  circleRef.value.style.left = `${scaledX}px`;
  circleRef.value.style.top = `${scaledY}px`;
});

// =========================
// LIFECYCLE
// =========================
onMounted(() => {
  if (!screenRef.value || !circleRef.value) return;

  // const screenWidth = screenRef.value.clientWidth;
  // const screenHeight = screenRef.value.clientHeight;

  // const circleWidth = circleRef.value.offsetWidth;
  // const circleHeight = circleRef.value.offsetHeight;

  setObjectPositionByLevel();

  // center it
  // circleXPosition.value = (screenWidth - circleWidth) / 2;
  // circleYPosition.value = (screenHeight - circleHeight) / 2;

  // circleRef.value.style.left = `${circleXPosition.value}px`;
  // circleRef.value.style.top = `${circleYPosition.value}px`;

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
  /* display: flex; */
  /* align-items: center;
  justify-content: center; */
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
</style>
