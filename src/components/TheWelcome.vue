<template>
  <div class="merge-game">
    <h2>⚙️ Merge Game — Item Generators</h2>

    <div class="board">
      <div
        v-for="(item, index) in items"
        :key="index"
        class="cell"
        :class="{ generator: item?.isGenerator }"
        draggable="true"
        @dragstart="item && onDragStart(index)"
        @dragover.prevent
        @drop="onDrop(index)"
        @click="item?.isGenerator && generateChild(index)"
      >
        <!-- Generator item -->
        <template v-if="item">
          <div class="item">
            <div class="emoji">
              {{ item.isGenerator ? "⚡" : emojiMap[item.type] }}
            </div>
            <div class="label">
              {{
                item.isGenerator
                  ? `GENERATOR: ${item.type.toUpperCase()}`
                  : `L${item.level}`
              }}
            </div>
          </div>
        </template>

        <!-- Empty cell -->
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

// 🪵 Helper: random integer
function randomInt(max: number): number {
  return Math.floor(Math.random() * max);
}

// 🧩 Populate the board with:
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
      const type = mergeItemTypes[randomInt(mergeItemTypes.length)];
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

// ⚡ When clicking a generator, spawn a new item
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

// 🧩 Drag logic
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

onMounted(() => populateItems());
</script>

<style scoped>
.board {
  display: grid;
  grid-template-columns: repeat(5, 80px);
  grid-template-rows: repeat(5, 80px);
  gap: 8px;
}

.cell {
  width: 80px;
  height: 80px;
  border: 2px solid #ccc;
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #fafafa;
  cursor: grab;
  user-select: none;
}

.cell.generator {
  border-color: #00bcd4;
  background: #e0f7fa;
  cursor: pointer;
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
  font-size: 12px;
  color: #555;
  text-align: center;
}
</style>
