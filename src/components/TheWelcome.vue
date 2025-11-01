<template>
  <div class="merge-game">
    <h2>⚙️ Merge Game — Item Generators</h2>

    <div class="board">
      <div
        v-for="(item, index) in items"
        :key="index"
        class="cell"
        :class="{ generator: item?.isGenerator, dragging: draggedIndex === index }"
        @pointerdown="startDrag(index)"
        @pointerup="drop(index)"
        @click="item?.isGenerator && generateChild(index)"
      >
        <template v-if="item">
          <div class="item">
            <div class="emoji">{{ item.isGenerator ? "⚡" : emojiMap[item.type] }}</div>
            <div class="label">
              {{
                item.isGenerator
                  ? `GENERATOR: ${item.type.toUpperCase()}`
                  : `L${item.level}`
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

function randomInt(max: number) {
  return Math.floor(Math.random() * max);
}

function populateItems() {
  const newItems: (MergeItem | null)[] = Array(BOARD_SIZE).fill(null);
  mergeItemTypes.forEach((type) => {
    let pos;
    do {
      pos = randomInt(BOARD_SIZE);
    } while (newItems[pos]);
    newItems[pos] = { id: Date.now() + pos, type, level: 0, isGenerator: true };
  });
  for (let i = 0; i < BOARD_SIZE; i++) {
    if (!newItems[i])
      newItems[i] = {
        id: Date.now() + i,
        type: mergeItemTypes[randomInt(mergeItemTypes.length)] as string,
        level: 1,
      };
  }
  items.value = newItems;
}

function findNearestEmptyIndex(fromIndex: number) {
  let nearestIndex: number | null = null;
  let minDistance = Infinity;
  items.value.forEach((slot, index) => {
    if (!slot) {
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

function generateChild(generatorIndex: number) {
  const generator = items.value[generatorIndex];
  if (!generator?.isGenerator) return;
  const emptyIndex = findNearestEmptyIndex(generatorIndex);
  if (emptyIndex === null) return;
  items.value[emptyIndex] = { id: Date.now(), type: generator.type, level: 1 };
}

function startDrag(index: number) {
  if (!items.value[index]) return;
  draggedIndex.value = index;
}

function drop(targetIndex: number) {
  if (draggedIndex.value === null || draggedIndex.value === targetIndex) return;

  const draggedItem = items.value[draggedIndex.value];
  const targetItem = items.value[targetIndex];
  if (!draggedItem) return;

  // Merge
  if (
    targetItem &&
    !targetItem.isGenerator &&
    !draggedItem.isGenerator &&
    draggedItem.type === targetItem.type &&
    draggedItem.level === targetItem.level
  ) {
    items.value[targetIndex] = {
      ...targetItem,
      level: targetItem.level + 1,
      id: Date.now(),
    };
    items.value[draggedIndex.value] = null;
  }
  // Move to empty
  else if (!targetItem) {
    items.value[targetIndex] = draggedItem;
    items.value[draggedIndex.value] = null;
  }
  // Swap
  else {
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
  user-select: none;
}
.cell.generator {
  border-color: #00bcd4;
  background: #e0f7fa;
}
.cell.dragging {
  opacity: 0.5;
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
