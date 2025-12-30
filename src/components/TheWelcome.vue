<template>
  <div class="merge-game">
    <h2>⚙️ Merge Game</h2>
    <Button label="Click Me" icon="pi pi-check" @click="reload" />
    <div>
      <h2>Energy: {{ energy }}⚡</h2>
      <h4 v-if="energy < 10">({{ formattedTotalCountdown }})</h4>
    </div>
    <div class="tasks-container">
      <ul class="task-unordered-list">
        <li v-for="(pair, index) in itemsToSubmit" class="task-list-item">
          <Button
            class="task-button"
            @click="submitItem(index)"
            :disabled="!isSubmitDisabled(pair, index)"
          >
            <!-- {{ isSubmitDisabled(pair, index) }} -->
            <img alt="pair_1" :src="getItemIcon(pair[0])" />
            <img alt="pair_2" :src="getItemIcon(pair[1])" />
          </Button>
        </li>
      </ul>
    </div>
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
              :class="{ pop: activeGenerators.has(index) }"
              @click.stop="item.isGenerator ? onGeneratorClick(index) : onCellTap(index)"
            >
              <template v-if="item.isGenerator">
                <div class="emoji">{{ getItemIcon(item) }}</div>
              </template>
              <template v-else>
                <img class="type-image" :src="getItemIcon(item)" alt="item" />
              </template>

              <span v-if="item.isGenerator" class="generator-overlay">⚡</span>
              <div v-if="maxLevelFlash.has(index)" class="max-overlay">MAX</div>
            </div>
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
            {{ draggedItem.isGenerator ? "⚡" : itemIconsMap[draggedItem.type] }}
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

    <div class="item-info">
      <div class="item-info-container">
        <div class="item-info-description" v-if="!isCellSelected">
          Tap an item to learn more
        </div>
        <div class="item-info-details" v-else>
          <div class="item-type-level">
            {{ isCellSelected.type.toUpperCase() }} -
            {{
              isCellSelected.isGenerator ? "Generator" : "Level " + isCellSelected.level
            }}
            <Badge
              class="item-info-badge"
              value="!"
              @click="() => (isItemModalOpen = true)"
            ></Badge>
          </div>
          <span class="item-merge-instruction"> Merge 2 same items to level up </span>
        </div>
      </div>
    </div>

    <Dialog
      v-model:visible="isItemModalOpen"
      modal
      :header="isCellSelected?.type + ' Info'"
      :style="{ width: '25rem' }"
    >
      <!-- TODO -->
    </Dialog>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed, watch } from "vue";
import type { Ref } from "vue";
import { Badge, Button, Dialog, InputText } from "primevue";

interface MergeItem {
  id: number;
  type: string;
  level: number;
  isGenerator?: boolean;
}

const generatorEmojiMap: Record<string, string> = {
  animal: "🌲",
  drinks: "🍹",
  clothes: "👕",
  food: "🍔",
  tools: "⚒️",
};

const itemIconsMap: Record<string, string[]> = {
  animal: [
    "animal_1",
    "animal_2",
    "animal_3",
    "animal_4",
    "animal_5",
    "animal_6",
    "animal_7",
  ],
  drinks: [
    "drinks_1",
    "drinks_2",
    "drinks_3",
    "drinks_4",
    "drinks_5",
    "drinks_6",
    "drinks_7",
    "drinks_8",
    "drinks_9",
  ],
  clothes: [
    "clothes_1",
    "clothes_2",
    "clothes_3",
    "clothes_4",
    "clothes_5",
    "clothes_6",
    "clothes_7",
  ],
  food: ["food_1", "food_2", "food_3", "food_4", "food_5"],
  tools: [
    "tools_1",
    "tools_2",
    "tools_3",
    "tools_4",
    "tools_5",
    "tools_6",
    "tools_7",
    "tools_8",
  ],
  // shape: ["shape_1", "shape_2", "shape_3", "shape_4", "shape_5", "shape_6"],
};

const energy = ref(10);
const countdown = ref(0); // time to next energy tick
const totalCountdown = ref(0); // FULL regen time countdown

let interval: number | undefined;

const mergeItemTypes = Object.keys(itemIconsMap);
const GRID_WIDTH = 7;
const GRID_HEIGHT = 9;
const BOARD_SIZE = GRID_WIDTH * GRID_HEIGHT; // 49
const MAX_LEVEL = 5;

const items = ref<(MergeItem | null)[]>(Array(BOARD_SIZE).fill(null));
const draggedIndex = ref<number | null>(null);
const selectedIndex = ref<number | null>(null);
const selectedCell = ref({
  item: null as MergeItem | null,
  index: null as number | null,
});

const mobileDragX = ref(0);
const mobileDragY = ref(0);
const maxLevelFlash = ref<Set<number>>(new Set());

const activeGenerators = ref<Set<number>>(new Set());
const iconFiles = import.meta.glob("../assets/icons/*.png", { eager: true, as: "url" });
const isCellSelected = ref<MergeItem | null>(null);
const isItemModalOpen = ref(false);

function getItemIcon(item: MergeItem): string {
  if (item.isGenerator) {
    const emoji = generatorEmojiMap[item.type];
    if (emoji) return emoji;
    return "⚙️";
  }

  const iconList = itemIconsMap[item.type];
  if (!iconList || iconList.length === 0) {
    console.warn(`⚠️ Missing icons for type: ${item.type}`);
    return iconFiles["../assets/icons/placeholder.png"] ?? "";
  }

  const levelIndex = Math.min(item.level - 1, iconList.length - 1);
  const filePath = `../assets/icons/${iconList[levelIndex]}.png`;

  return iconFiles[filePath] ?? iconFiles["../assets/icons/placeholder.png"] ?? "";
}

function onGeneratorClick(index: number) {
  if (!energy.value) return;
  energy.value--;
  console.log("energy:", energy.value);

  generateChild(index); // only generate one item

  // Trigger pop animation
  activeGenerators.value.add(index);
  setTimeout(() => activeGenerators.value.delete(index), 300);
}

function getCellColor(index: number): string {
  const cols = 5;
  const row = Math.floor(index / cols);
  const col = index % cols;
  return (row + col) % 2 === 0 ? "light-cell" : "dark-cell";
}

function onCellTap(index: number) {
  const item = items.value[index];

  if (!item) return;
  selectedCell.value.item = { ...item };
  selectedCell.value.index = index;

  if (item?.isGenerator) {
    return;
  } else {
    console.log("cell clicked", index);
    isCellSelected.value = item;
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

  // Step 3: place random items with weighted levels
  let placed = 0;
  while (placed < extraItemsCount) {
    const pos = randomInt(BOARD_SIZE);
    if (!newItems[pos]) {
      const type = mergeItemTypes[randomInt(mergeItemTypes.length)] as string;

      // Determine level using 70% → 1, 25% → 2, 5% → 3
      const rand = Math.random();
      let level: number;
      if (rand < 0.7) {
        level = 1;
      } else if (rand < 0.95) {
        level = 2;
      } else {
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

const formattedTotalCountdown = computed(() => {
  const minutes = Math.floor(totalCountdown.value / 60);
  const seconds = totalCountdown.value % 60;
  return `${String(minutes).padStart(2, "0")}:${String(seconds).padStart(2, "0")}`;
});

const startEnergyRegen = () => {
  if (interval !== undefined) clearInterval(interval);

  const missing = 10 - energy.value;
  if (missing <= 0) return;

  // FULL time remaining (10 sec per energy)
  totalCountdown.value = missing * 10;

  countdown.value = 10; // next energy tick

  interval = (setInterval(() => {
    // Decrease total time
    totalCountdown.value--;

    // Decrease next-energy timer
    countdown.value--;

    // When countdown hits 0 → gain energy
    if (countdown.value === 0) {
      energy.value++;

      if (energy.value >= 10) {
        // Finished
        clearInterval(interval);
        interval = undefined;
        return;
      }

      countdown.value = 10; // reset for next energy
    }
  }, 1000) as unknown) as number;
};

const itemsToSubmit: Ref<[MergeItem, MergeItem][]> = ref([]);

function submitItem(index: number) {
  if (!indexToSubmitFound.value) return;

  indexToSubmitFound.value.forEach((item) => {
    if (item.listIndex === index) {
      items.value[item.itemIndexes[0]] = null;
      items.value[item.itemIndexes[1]] = null;

      itemsToSubmit.value?.splice(index, 1);
    }
  });
}

function createRandomItemPairs(): [MergeItem, MergeItem][] {
  const PAIR_COUNT = 3;
  const MAX_LEVEL = 5;

  return Array.from({ length: PAIR_COUNT }, (_, pairIndex) => {
    const createRandomItem = (offset: number): MergeItem => ({
      id: Date.now() + pairIndex * 10 + offset,
      type: mergeItemTypes[randomInt(mergeItemTypes.length)] as string,
      level: randomInt(MAX_LEVEL) + 1, // 1–3 only
    });

    return [createRandomItem(0), createRandomItem(1)];
  });
}

function reload() {
  window.location.reload();
}

const indexToSubmitFound = ref<
  [
    {
      listIndex: number;
      itemIndexes: [number, number];
    }
  ]
>([] as any);

function findPairIndexes(
  pair: [MergeItem, MergeItem],
  index: number
): [number, number] | null {
  const indexes: number[] = [];

  let matchesPair0 = false;
  let matchesPair1 = false;
  for (let i = 0; i < items.value.length; i++) {
    const item = items.value[i];

    if (item && !item.isGenerator) {
      matchesPair0 = item.type === pair[0].type && item.level === pair[0].level;

      if (matchesPair0) {
        indexes.push(i);
        break;
      }
    }
  }

  if (matchesPair0) {
    for (let i = 0; i < items.value.length; i++) {
      const item = items.value[i];

      if (item && !item.isGenerator) {
        const matchesPair1 = item.type === pair[1].type && item.level === pair[1].level;

        if (matchesPair1 && indexes[0] !== i) {
          indexes.push(i);

          if (indexes.length < 3) {
            indexToSubmitFound.value.push({
              listIndex: index,
              itemIndexes: [indexes[0]!, indexes[1]!],
            });
          }
          break;
        }
      }
    }
  }

  if (!matchesPair0 && !matchesPair1) return null;

  return indexes.length === 2 ? [indexes[0]!, indexes[1]!] : null;
}

function isSubmitDisabled(pair: [MergeItem, MergeItem], index: number) {
  return findPairIndexes(pair, index);
}

watch(energy, (newVal) => {
  if (newVal <= 10) {
    startEnergyRegen();
  }
});

watch(
  itemsToSubmit,
  (newPairs) => {
    if (!newPairs) return;
    if (newPairs.length === 3) return;
    const PAIR_COUNT = 3;
    const MAX_LEVEL = 3;

    while (itemsToSubmit.value.length < PAIR_COUNT) {
      const pairIndex = itemsToSubmit.value.length;

      const createRandomItem = (offset: number): MergeItem => ({
        id: Date.now() + pairIndex * 10 + offset,
        type: mergeItemTypes[randomInt(mergeItemTypes.length)] as string,
        level: randomInt(MAX_LEVEL) + 1, // 1–3
      });
      break;

      itemsToSubmit.value.push([createRandomItem(0), createRandomItem(1)]);
    }
  },
  { deep: true, immediate: true }
);

onMounted(() => {
  isMobile.value = "ontouchstart" in window || navigator.maxTouchPoints > 0;
  populateItems();
  itemsToSubmit.value = createRandomItemPairs();
});
</script>

<style scoped>
.item-info {
  margin-block: 20px;
  padding-block: 10px;
  display: flex;
  align-items: center;
  justify-content: center;

  .item-info-details {
    display: grid;
    align-items: center;
    justify-content: center;
    background-color: #ccc;
    padding: 15px;
    border-radius: 10px;
    width: 400px;

    .item-type-level {
      font-size: 20px;
      color: black;

      .item-info-badge {
        position: relative;
        top: -4px;
        cursor: pointer;
      }
    }

    .item-merge-instruction {
      color: black;
    }
  }
}
.merge-game {
  display: grid;
  justify-content: center;
  margin: 12px;
  padding: 12px;
}

.board {
  display: grid;
  grid-template-columns: repeat(7, 70px); /* 7 columns */
  grid-template-rows: repeat(9, 70px); /* 7 rows */
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
  pointer-events: none;
  z-index: 9999;
  opacity: 0.8;
  transform: translate(-50%, -50%);
}

.item {
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative; /* needed for overlay */
}

.emoji {
  font-size: 2rem;
  transition: all 0.3s ease;
  filter: saturate(1.2) contrast(1) brightness(1.1);
}

/* Pop effect when clicked */
.emoji.pop {
  transform: scale(1.3);
  text-shadow: 2px 2px 0 #ccc, 4px 4px 4px rgba(0, 0, 0, 0.4);
}

/* small lightning icon for all generators */
.generator-overlay {
  position: absolute;
  font-size: 10px; /* smaller than main emoji */
  bottom: 0;
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

.emoji-wrapper.pop .emoji {
  animation: generator-pop 0.3s ease-out forwards;
}

@keyframes generator-pop {
  0% {
    transform: scale(1);
    /* filter: drop-shadow(0 0 0 #fff176); */
  }
  50% {
    transform: scale(1.5);
    /* filter: drop-shadow(0 0 10px #fff176) drop-shadow(0 0 20px #fff176); */
  }
  100% {
    transform: scale(1);
    /* filter: drop-shadow(0 0 0 #fff176); */
  }
}

.tasks-container {
  margin-block: 10px;
  /* width: 1000px; */
  max-width: 560px;
  overflow-x: auto;
  overflow-y: hidden;
  height: 80px;

  .task-unordered-list {
    display: flex;
    flex-direction: row;
    gap: 8px;

    padding: 0;
    margin: 0;
    list-style: none;
  }

  .task-list-item {
    float: left;
    flex: 0 0 auto;
  }

  .task-button {
    margin-right: 10px;
  }
}
</style>
