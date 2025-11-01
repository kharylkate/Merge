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
          <div class="item">
            <div class="emoji">
              {{ item.isGenerator ? "⚡" : emojiMap[item.type] }}
            </div>
            <div class="label">
              {{
                item.isGenerator ? `PRODUCE ${item.type.toUpperCase()}` : `L${item.level}`
              }}
            </div>
          </div>
        </template>

        <div v-else class="empty">•</div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue";

interface MergeItem {
  id: number;
  type: string;
  level: number;
  isGenerator?: boolean;
}

const emojiMap: Record<string, string> = {
  wood: "🌲",
  stone: "🪨",
  iron: "⚙️",
  gold: "💰",
  crystal: "💎",
};

const mergeItemTypes = Object.keys(emojiMap);
const BOARD_SIZE = 25;
const GRID_WIDTH = 5;

const items = ref<(MergeItem | null)[]>(Array(BOARD_SIZE).fill(null));
const draggedIndex = ref<number | null>(null);
const selectedIndex = ref<number | null>(null);

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

  // Step 2: fill the remaining spaces with random level 1 items
  for (let i = 0; i < BOARD_SIZE; i++) {
    if (!newItems[i]) {
      const type = mergeItemTypes[randomInt(mergeItemTypes.length)] as string;
      newItems[i] = {
        id: Date.now() + i,
        type,
        level: 1,
      };
    }
  }

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
    const mergedItem: MergeItem = {
      id: Date.now(),
      type: targetItem.type,
      level: targetItem.level + 1,
    };
    items.value[targetIndex] = mergedItem;
    items.value[draggedIndex.value] = null;
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

  // Only start dragging if finger moved enough
  if (dx > tapThreshold || dy > tapThreshold) {
    mobileDrag.value.isDragging = true;
  }

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
  grid-template-columns: repeat(5, 80px);
  grid-template-rows: repeat(5, 80px);
  /* gap: 4px; */
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

.item {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.empty {
  opacity: 0.3;
  font-size: 20px;
  color: #bbb;
}

.emoji {
  font-size: 32px;
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
  border-color: #00bcd4;
  background-color: #e0f7fa !important;
  cursor: pointer;
}
</style>
