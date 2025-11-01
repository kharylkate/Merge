<template>
  <div class="merge-game">
    <h2>⚙️ Merge Game — Item Generators</h2>

    <div class="board">
      <div
        v-for="(item, index) in items"
        :key="index"
        :data-index="index"
        :class="[
          'cell',
          getCellColor(index),
          { generator: item?.isGenerator, selected: selectedIndex === index },
        ]"
        @click="onCellTap(index)"
        :draggable="!isMobile"
        @dragstart="!isMobile && item ? onDragStart(index) : null"
        @dragover.prevent="!isMobile && true"
        @drop="!isMobile ? onDrop(index) : null"
        @touchstart.prevent="isMobile ? onTouchStart(index, $event) : null"
        @touchmove.prevent="isMobile ? onTouchMove($event) : null"
        @touchend.prevent="isMobile ? onTouchEnd() : null"
      >
        <template v-if="item">
          <div class="item" :class="{ generator: item.isGenerator }">
            <div
              class="emoji-wrapper"
              :class="{ 'animate-max': maxLevelFlash.has(index) }"
            >
              <div class="emoji">
                {{
                  item.isGenerator
                    ? generatorEmojiMap[item.type]
                    : emojiMap[item.type]?.[item.level - 1] ?? "❓"
                }}
              </div>
              <div v-if="maxLevelFlash.has(index)" class="max-overlay">MAX</div>
              <span v-if="item.isGenerator" class="generator-overlay">⚡</span>
            </div>
            <!-- <div class="emoji">
              {{
                item.isGenerator
                  ? generatorEmojiMap[item.type]
                  : emojiMap[item.type]?.[item.level - 1] ?? "❓"
              }}
              <span v-if="item.isGenerator" class="generator-overlay">⚡</span>
            </div> -->
          </div>
        </template>

        <div v-else class="empty">•</div>
      </div>

      <div
        v-if="mobileDrag.isDragging && mobileDrag.startIndex !== null"
        class="mobile-drag-copy"
        :style="{
          top: mobileDragY + 'px',
          left: mobileDragX + 'px',
        }"
      >
        <div class="item" v-if="draggedItem">
          <div class="emoji">
            {{ draggedItem.isGenerator ? "⚡" : emojiMap[draggedItem.type] }}
          </div>
          <div class="label">
            {{
              draggedItem.isGenerator
                ? `PRODUCE ${draggedItem.type.toUpperCase()}`
                : `L${draggedItem.level}`
            }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from "vue";

interface MergeItem {
  id: number;
  type: string;
  level: number;
  isGenerator?: boolean;
}

const emojiMap: Record<string, string[]> = {
  wood: ["🌱", "🌿", "🌳", "🎋", "🌲"], // level 1 → 5
  stone: ["🪵", "🪨", "🗿", "⛰️", "🏔️"],
  iron: ["🔩", "⚙️", "🛠️", "🪛", "⚒️"],
  gold: ["🪙", "💰", "🏆", "🥇", "💎"],
  crystal: ["🔹", "🔷", "💠", "✨", "💎"],
};

const generatorEmojiMap: Record<string, string> = {
  wood: "🌳", // generic tree for wood generator
  stone: "⛰️", // mountain/stone for stone generator
  iron: "⚙️", // gear for iron generator
  gold: "💰", // money bag for gold generator
  crystal: "💎", // gem for crystal generator
};

const mergeItemTypes = Object.keys(emojiMap);
const GRID_WIDTH = 5;
const GRID_HEIGHT = 7;
const BOARD_SIZE = GRID_WIDTH * GRID_HEIGHT; // 40
const MAX_LEVEL = 5;

const items = ref<(MergeItem | null)[]>(Array(BOARD_SIZE).fill(null));
const draggedIndex = ref<number | null>(null);
const selectedIndex = ref<number | null>(null);

const mobileDragX = ref(0);
const mobileDragY = ref(0);
const maxLevelFlash = ref<Set<number>>(new Set());

function getCellColor(index: number): string {
  const cols = 5;
  const row = Math.floor(index / cols);
  const col = index % cols;
  return (row + col) % 2 === 0 ? "light-cell" : "dark-cell";
}

function onCellTap(index: number) {
  const item = items.value[index];

  if (item?.isGenerator) {
    generateChild(index);
  } else {
    console.log("cell clicked", index);
    // optionally: select for mobile merge if needed
    if (selectedIndex.value === null && item) {
      selectedIndex.value = index;
    } else if (selectedIndex.value !== null) {
      onDrop(index);
      selectedIndex.value = null;
    }
  }
}

// Helper: random integer
function randomInt(max: number): number {
  return Math.floor(Math.random() * max);
}

// Populate the board with:
// - 1 generator per item type
// - Random merge items for the rest
function populateItems(): void {
  const newItems: (MergeItem | null)[] = Array(BOARD_SIZE).fill(null);

  // Step 1: place one generator per item type
  const generatorPositions: number[] = [];
  mergeItemTypes.forEach((type) => {
    let pos: number;
    do {
      pos = randomInt(BOARD_SIZE);
    } while (newItems[pos] !== null);
    newItems[pos] = {
      id: Date.now() + pos,
      type,
      level: 0,
      isGenerator: true,
    };
    generatorPositions.push(pos);
  });

  // Step 2: determine how many extra items to place
  const maxFilled = Math.floor(BOARD_SIZE * 0.6); // max 60% filled
  const extraItemsCount = maxFilled - mergeItemTypes.length; // minus generators

  // Step 3: place random items with 80% chance for level 1 or 2, 20% for level 3
  let placed = 0;
  while (placed < extraItemsCount) {
    const pos = randomInt(BOARD_SIZE);
    if (!newItems[pos]) {
      const type = mergeItemTypes[randomInt(mergeItemTypes.length)] as string;

      // Determine level
      const rand = Math.random(); // 0 → 1
      let level: number;
      if (rand < 0.8) {
        // 80% chance → level 1 or 2
        level = Math.random() < 0.5 ? 1 : 2;
      } else {
        // 20% chance → level 3
        level = 3;
      }

      newItems[pos] = {
        id: Date.now() + pos + placed,
        type,
        level,
      };
      placed++;
    }
  }

  // Remaining cells stay null (empty)
  items.value = newItems;
}

// 🔹 Find nearest empty space to a given index
function findNearestEmptyIndex(fromIndex: number): number | null {
  let nearestIndex: number | null = null;
  let minDistance = Infinity;

  items.value.forEach((slot, index) => {
    if (slot === null) {
      const dx = (index % GRID_WIDTH) - (fromIndex % GRID_WIDTH);
      const dy = Math.floor(index / GRID_WIDTH) - Math.floor(fromIndex / GRID_WIDTH);
      const distance = Math.sqrt(dx * dx + dy * dy);
      if (distance < minDistance) {
        minDistance = distance;
        nearestIndex = index;
      }
    }
  });

  return nearestIndex;
}

// When clicking a generator, spawn a new item
function generateChild(generatorIndex: number): void {
  const generator = items.value[generatorIndex];
  if (!generator || !generator.isGenerator) return;

  const emptyIndex = findNearestEmptyIndex(generatorIndex);
  if (emptyIndex === null) {
    console.log("No empty space available!");
    return;
  }

  const newItem: MergeItem = {
    id: Date.now(),
    type: generator.type,
    level: 1,
  };

  items.value[emptyIndex] = newItem;
}

// Drag logic
function onDragStart(index: number): void {
  draggedIndex.value = index;
}

function onDrop(targetIndex: number): void {
  if (draggedIndex.value === null || draggedIndex.value === targetIndex) return;

  const draggedItem = items.value[draggedIndex.value];
  const targetItem = items.value[targetIndex];
  if (!draggedItem) return;

  // Case 1: Merge — same type + same level, target is NOT a generator
  if (
    targetItem &&
    !targetItem.isGenerator &&
    !draggedItem.isGenerator &&
    draggedItem.type === targetItem.type &&
    draggedItem.level === targetItem.level
  ) {
    if (targetItem.level >= MAX_LEVEL) {
      // Already max level → flash MAX indicator
      maxLevelFlash.value.add(targetIndex);
      setTimeout(() => {
        maxLevelFlash.value.delete(targetIndex);
      }, 1000); // show for 1 second
    } else {
      // Normal merge
      const mergedItem: MergeItem = {
        id: Date.now(),
        type: targetItem.type,
        level: targetItem.level + 1,
      };
      items.value[targetIndex] = mergedItem;
      items.value[draggedIndex.value] = null;
    }
  }
  // Case 2: Move to empty
  else if (!targetItem) {
    items.value[targetIndex] = draggedItem;
    items.value[draggedIndex.value] = null;
  }
  // Case 3: Swap — target exists and is either a different item OR a generator
  else if (
    targetItem &&
    // Allow swap if dragged item is different OR target is a generator
    (targetItem.isGenerator ||
      draggedItem.type !== targetItem.type ||
      draggedItem.level !== targetItem.level)
  ) {
    items.value[targetIndex] = draggedItem;
    items.value[draggedIndex.value] = targetItem;
  }

  draggedIndex.value = null;
}

function onCellClick(index: number) {
  const item = items.value[index];

  if (item?.isGenerator) {
    generateChild(index);
    return;
  }

  // Mobile drag selection: tap to select, then tap to drop
  if (selectedIndex.value === null && item) {
    selectedIndex.value = index;
  } else if (selectedIndex.value !== null) {
    onDrop(index);
    selectedIndex.value = null;
  }
}

let touchStartIndex: number | null = null;

function onTouchStart(index: number, event: TouchEvent) {
  const touch = event.touches[0];
  if (!touch) return;

  touchStartX = touch.clientX;
  touchStartY = touch.clientY;

  mobileDrag.value.startIndex = index;
  mobileDrag.value.currentIndex = index;
  mobileDrag.value.isDragging = false;
}

function onTouchMove(event: TouchEvent) {
  if (mobileDrag.value.startIndex === null) return;

  const touch = event.touches[0];
  if (!touch) return;

  const dx = Math.abs(touch.clientX - touchStartX);
  const dy = Math.abs(touch.clientY - touchStartY);

  if (dx > tapThreshold || dy > tapThreshold) {
    mobileDrag.value.isDragging = true;
  }

  // Update the floating copy position
  mobileDragX.value = touch.clientX;
  mobileDragY.value = touch.clientY;

  // Track the cell under the finger
  const el = document.elementFromPoint(touch.clientX, touch.clientY) as HTMLElement;
  if (!el) return;

  const cellEl = el.closest(".cell") as HTMLElement;
  if (!cellEl) return;

  const indexAttr = cellEl.getAttribute("data-index");
  if (!indexAttr) return;

  mobileDrag.value.currentIndex = parseInt(indexAttr, 10);
}

function onTouchEnd() {
  const { startIndex, currentIndex, isDragging } = mobileDrag.value;

  if (startIndex === null) return;

  if (!isDragging) {
    // Tap → select / generator
    onCellTap(startIndex);
  } else if (currentIndex !== null && startIndex !== currentIndex) {
    // Drag → drop / merge
    draggedIndex.value = startIndex;
    onDrop(currentIndex);
  }

  // Reset
  mobileDrag.value.startIndex = null;
  mobileDrag.value.currentIndex = null;
  mobileDrag.value.isDragging = false;
}

const mobileDrag = ref<{
  startIndex: number | null; // initial touched cell
  currentIndex: number | null; // cell currently under finger
  isDragging: boolean;
}>({ startIndex: null, currentIndex: null, isDragging: false });

const tapThreshold = 10; // pixels to distinguish tap vs drag
let touchStartX = 0;
let touchStartY = 0;

const isMobile = ref(false);

function getDraggedItem(): MergeItem | null {
  if (mobileDrag.value.startIndex === null) return null;

  const item = items.value[mobileDrag.value.startIndex];
  return item ?? null; // convert undefined to null
}

const draggedItem = computed(() => getDraggedItem());

onMounted(() => {
  isMobile.value = "ontouchstart" in window || navigator.maxTouchPoints > 0;
  populateItems();
});
</script>

<style scoped>
.merge-game {
  display: grid;
  justify-content: center;
  margin: 12px;
  padding: 12px;
}

.board {
  display: grid;
  grid-template-columns: repeat(5, 80px); /* 5 columns */
  grid-template-rows: repeat(7, 80px); /* 8 rows */
  /* gap: 4px; optional spacing between cells */
  background: #e6d8b3;
  border-radius: 5px;
}

.cell {
  width: 80px;
  height: 80px;
  border: 0px;
  /* border: 1spx solid #ccc; */
  /* border-radius: 10px; */
  display: flex;
  align-items: center;
  justify-content: center;
  background: #fafafa;
  cursor: grab;
  user-select: none;
}

.empty {
  opacity: 0.3;
  font-size: 20px;
  color: #bbb;
}

.label {
  font-size: 10px;
  color: #555;
  text-align: center;
}

.light-cell {
  background-color: #e6d8b3;
}

.dark-cell {
  background-color: #d6c7a1;
}

.cell.generator {
  /* border-color: #00bcd4; */
  /* background-color: #e0f7fa !important; */
  cursor: pointer;
}

.mobile-drag-copy {
  position: fixed;
  width: 80px;
  height: 80px;
  pointer-events: none; /* so it doesn’t block touches */
  z-index: 9999;
  opacity: 0.8;
  transform: translate(-50%, -50%); /* CENTER the div at top/left coordinates */
}

.item {
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative; /* needed for overlay */
}

.emoji {
  font-size: 32px;
  position: relative;
}

/* small lightning icon for all generators */
.generator-overlay {
  position: absolute;
  font-size: 10px; /* smaller than main emoji */
  bottom: 0; /* bottom right corner */
  right: 0;
  background: #fff;
  border-radius: 50%;
  padding: 1px 2px;
  box-shadow: 0 0 2px rgba(0, 0, 0, 0.3);
}

.emoji-wrapper {
  position: relative;
  display: inline-block;
}

.max-overlay {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(255, 0, 0, 0.8);
  color: white;
  font-weight: bold;
  padding: 2px 6px;
  border-radius: 4px;
  font-size: 14px;
  pointer-events: none;
  z-index: 10;
}

@keyframes pop-shake {
  0% {
    transform: scale(1) rotate(0deg);
  }
  25% {
    transform: scale(1.2) rotate(-10deg);
  }
  50% {
    transform: scale(1.1) rotate(10deg);
  }
  75% {
    transform: scale(1.2) rotate(-5deg);
  }
  100% {
    transform: scale(1) rotate(0deg);
  }
}

.animate-max {
  animation: pop-shake 0.5s ease-in-out;
}
</style>
