<template>
  <div class="object-detection-marker">
    <!-- Main canvas container -->
    <div class="canvas-container">
      <canvas
        ref="canvasRef"
        class="marker-canvas"
        :width="canvasWidth"
        :height="canvasHeight"
        @mousedown="handleMouseDown"
        @mousemove="handleMouseMove"
        @mouseup="handleMouseUp"
        @mouseleave="handleMouseLeave"
        @touchstart.prevent="handleTouchStart"
        @touchmove.prevent="handleTouchMove"
        @touchend.prevent="handleTouchEnd"
      />

      <!-- Loading indicator -->
      <div v-if="!imageLoaded && props.image" class="loading-indicator">
        이미지를 로딩 중...
      </div>

      <!-- Error message -->
      <div v-if="errorMessage" class="error-message">
        {{ errorMessage }}
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import {
  ref,
  computed,
  watch,
  onMounted,
  onUnmounted,
  withDefaults,
  defineProps,
  defineEmits,
  defineExpose,
} from 'vue';
import type {
  Props,
  SelectionMode,
  GridCell,
  ColorLayer,
  LayerSelectionData,
  ExtendedSelectionData,
  ColorLayerExport,
  BrushShape,
  GridLayerExport,
  Rectangle,
} from '../types';
import {
  calculateGridFromResolution,
  screenToGridResolution,
  getGridKey,
  mergeGridsToRects,
} from '../utils';

// --- Props and Emits ---
const props = withDefaults(defineProps<Props>(), {
  resolution: 1,
  selectionMode: 'point',
  highlightColor: '#007bff',
  gridColor: '#cccccc',
  layerColors: () => [
    '#FF0000',
    '#00FF00',
    '#0000FF',
    '#FFFF00',
    '#FF00FF',
    '#00FFFF',
  ],
  canvasWidth: 800,
  canvasHeight: 600,
  hoverColor: '#6c757d',
  backgroundColor: '#ffffff',
  defaultBrushSize: 1,
  defaultBrushShape: 'circle',
});

const emit = defineEmits<{
  selectionChange: [selection: ExtendedSelectionData];
  modeChange: [mode: SelectionMode];
  imageLoad: [image: HTMLImageElement];
  imageError: [error: Error];
  gridHover: [cell: GridCell | null];
  layerChange: [layerData: LayerSelectionData];
  activeLayerChange: [color: string];
  layerVisibilityChange: [color: string, visible: boolean];
  layerAdded: [color: string, layer: ColorLayer];
  layerRemoved: [color: string];
}>();

// --- Reactive State ---
const canvasRef = ref<HTMLCanvasElement | null>(null);
const currentMode = ref<SelectionMode>(props.selectionMode || 'point');
const errorMessage = ref<string>('');

// Image State
const imageLoaded = ref<boolean>(false);
const imageElement = ref<HTMLImageElement | null>(null);

// Grid State
const imageAspectRatio = ref<{ width: number; height: number } | null>(null);
const gridDimensions = ref<{ cols: number; rows: number } | null>(null);
const cellSize = ref<{ width: number; height: number } | null>(null);

// Selection State
const isSelecting = ref<boolean>(false);
const hoveredCell = ref<GridCell | null>(null);
const lastSelectedCell = ref<GridCell | null>(null);
const selectionStart = ref<GridCell | null>(null);
const selectionEnd = ref<GridCell | null>(null);

// Color Layer State
const colorLayers = ref<Map<string, ColorLayer>>(new Map());
const activeColorLayer = ref<string>('');

// Brush State
const brushSize = ref(1);
const brushShape = ref<BrushShape>('circle');

// --- Computed Properties ---
const canvasWidth = computed(() => props.canvasWidth || 800);
const canvasHeight = computed(() => props.canvasHeight || 600);
const canvasContext = computed(() => canvasRef.value?.getContext('2d') || null);

// --- Helper Functions ---
const hexToRgba = (hex: string, alpha: number): string => {
  if (!hex || !hex.startsWith('#')) return `rgba(0, 0, 0, ${alpha})`;
  const r = parseInt(hex.slice(1, 3), 16) || 0;
  const g = parseInt(hex.slice(3, 5), 16) || 0;
  const b = parseInt(hex.slice(5, 7), 16) || 0;
  return `rgba(${r}, ${g}, ${b}, ${alpha})`;
};

const getColorName = (color: string): string => {
  const colorNames: Record<string, string> = {
    '#FF0000': '빨강',
    '#00FF00': '초록',
    '#0000FF': '파랑',
    '#FFFF00': '노랑',
    '#FF00FF': '마젠타',
    '#00FFFF': '청록',
  };
  return colorNames[color.toUpperCase()] || color;
};

// --- Core Methods ---
const initColorLayers = () => {
  colorLayers.value.clear();
  const colors = props.layerColors || [];
  colors.forEach((color: string) => {
    colorLayers.value.set(color.toUpperCase(), {
      color: color.toUpperCase(),
      selectedGrids: new Set<string>(),
      visible: true,
      name: getColorName(color),
      opacity: 0.5,
    });
  });
  if (colors.length > 0) {
    activeColorLayer.value = colors[0].toUpperCase();
  }
};

const switchMode = (mode: SelectionMode) => {
  currentMode.value = mode;
  emit('modeChange', mode);
  if (mode !== 'rectangle') {
    selectionStart.value = null;
    selectionEnd.value = null;
  }
  redrawCanvas();
};

// --- Drawing Methods ---
const redrawCanvas = () => {
  const ctx = canvasContext.value;
  if (!ctx) return;

  ctx.fillStyle = props.backgroundColor;
  ctx.fillRect(0, 0, canvasWidth.value, canvasHeight.value);

  if (imageElement.value && imageLoaded.value) {
    ctx.drawImage(
      imageElement.value,
      0,
      0,
      canvasWidth.value,
      canvasHeight.value
    );
  }

  drawGrid(ctx);
  drawColorLayers(ctx);

  if (hoveredCell.value) {
    const activeLayer = colorLayers.value.get(activeColorLayer.value);
    const hoverColor = activeLayer ? activeLayer.color : props.hoverColor;
    drawCellHighlight(ctx, hoveredCell.value, hoverColor, 0.25);
  }

  // Draw brush preview
  if (
    hoveredCell.value &&
    (currentMode.value === 'point' || currentMode.value === 'eraser')
  ) {
    drawBrushPreview(ctx);
  }

  // Draw rectangle selection preview (for rectangle mode)
  if (
    currentMode.value === 'rectangle' &&
    selectionStart.value &&
    selectionEnd.value
  ) {
    drawRectanglePreview(ctx);
  }
};

const drawGrid = (ctx: CanvasRenderingContext2D) => {
  if (!cellSize.value) return;
  ctx.strokeStyle = props.gridColor;
  ctx.lineWidth = 1;
  for (let x = 0; x <= canvasWidth.value; x += cellSize.value.width) {
    ctx.beginPath();
    ctx.moveTo(x, 0);
    ctx.lineTo(x, canvasHeight.value);
    ctx.stroke();
  }
  for (let y = 0; y <= canvasHeight.value; y += cellSize.value.height) {
    ctx.beginPath();
    ctx.moveTo(0, y);
    ctx.lineTo(canvasWidth.value, y);
    ctx.stroke();
  }
};

const drawColorLayers = (ctx: CanvasRenderingContext2D) => {
  colorLayers.value.forEach((layer: ColorLayer) => {
    if (layer.visible) {
      layer.selectedGrids.forEach((key: string) => {
        const [row, col] = key.split(',').map(Number);
        drawCellHighlight(ctx, { row, col }, layer.color, layer.opacity);
      });
    }
  });
};

const drawCellHighlight = (
  ctx: CanvasRenderingContext2D,
  cell: GridCell,
  color: string,
  fillOpacity: number = 0.5
) => {
  if (!cellSize.value) return;
  const x = cell.col * cellSize.value.width;
  const y = cell.row * cellSize.value.height;
  ctx.fillStyle = hexToRgba(color, fillOpacity);
  ctx.fillRect(x, y, cellSize.value.width, cellSize.value.height);
  ctx.strokeStyle = color;
  ctx.lineWidth = 2;
  ctx.strokeRect(x, y, cellSize.value.width, cellSize.value.height);
};

const drawRectanglePreview = (ctx: CanvasRenderingContext2D) => {
  if (!selectionStart.value || !selectionEnd.value || !cellSize.value) return;
  const startX =
    Math.min(selectionStart.value.col, selectionEnd.value.col) *
    cellSize.value.width;
  const startY =
    Math.min(selectionStart.value.row, selectionEnd.value.row) *
    cellSize.value.height;
  const endX =
    (Math.max(selectionStart.value.col, selectionEnd.value.col) + 1) *
    cellSize.value.width;
  const endY =
    (Math.max(selectionStart.value.row, selectionEnd.value.row) + 1) *
    cellSize.value.height;

  const activeLayer = colorLayers.value.get(activeColorLayer.value);
  const previewColor = activeLayer ? activeLayer.color : props.highlightColor;

  ctx.strokeStyle = previewColor;
  ctx.lineWidth = 2;
  ctx.setLineDash([5, 5]);
  ctx.strokeRect(startX, startY, endX - startX, endY - startY);
  ctx.setLineDash([]);
};

const drawBrushPreview = (ctx: CanvasRenderingContext2D) => {
  if (!hoveredCell.value || !cellSize.value) return;

  const { row, col } = hoveredCell.value;
  const { width, height } = cellSize.value;
  const centerX = col * width + width / 2;
  const centerY = row * height + height / 2;

  const activeLayer = colorLayers.value.get(activeColorLayer.value);
  const previewColor = activeLayer ? activeLayer.color : props.highlightColor;

  ctx.strokeStyle = previewColor;
  ctx.lineWidth = 2;
  ctx.setLineDash([3, 3]);

  if (brushShape.value === 'circle') {
    const radius = (brushSize.value - 0.5) * Math.min(width, height);
    ctx.beginPath();
    ctx.arc(centerX, centerY, radius, 0, Math.PI * 2);
    ctx.stroke();
  } else {
    // Square
    const size = brushSize.value * 2 - 1;
    const previewWidth = size * width;
    const previewHeight = size * height;
    ctx.strokeRect(
      centerX - previewWidth / 2,
      centerY - previewHeight / 2,
      previewWidth,
      previewHeight
    );
  }

  ctx.setLineDash([]);
};

// --- Event Handlers ---
const handleMouseDown = (event: MouseEvent) => {
  event.preventDefault();
  const cell = getGridCellFromCoordinates(event.clientX, event.clientY);
  if (!cell) return;

  isSelecting.value = true;

  if (currentMode.value === 'rectangle') {
    selectionStart.value = cell;
    selectionEnd.value = cell;
  } else {
    const operation = currentMode.value === 'eraser' ? 'deselect' : 'select';
    toggleGridSelection(cell, operation);
    lastSelectedCell.value = cell;
  }

  emitSelectionChange();
  redrawCanvas();
};

const handleMouseMove = (event: MouseEvent) => {
  const cell = getGridCellFromCoordinates(event.clientX, event.clientY);

  if (
    hoveredCell.value?.row !== cell?.row ||
    hoveredCell.value?.col !== cell?.col
  ) {
    hoveredCell.value = cell;
    emit('gridHover', cell);
    redrawCanvas(); // Redraw to update hover preview
  }

  if (!isSelecting.value || !cell) return;

  if (currentMode.value === 'rectangle') {
    selectionEnd.value = cell;
    redrawCanvas();
  } else {
    const operation = currentMode.value === 'eraser' ? 'deselect' : 'select';
    if (
      !lastSelectedCell.value ||
      lastSelectedCell.value.row !== cell.row ||
      lastSelectedCell.value.col !== cell.col
    ) {
      toggleGridSelection(cell, operation);
      lastSelectedCell.value = cell;

      // Update canvas and emit changes after brush/single cell operation
      emitSelectionChange();
      redrawCanvas();
    }
  }
};

const handleMouseUp = () => {
  if (!isSelecting.value) return;
  isSelecting.value = false;

  if (
    currentMode.value === 'rectangle' &&
    selectionStart.value &&
    selectionEnd.value
  ) {
    const minRow = Math.min(selectionStart.value.row, selectionEnd.value.row);
    const maxRow = Math.max(selectionStart.value.row, selectionEnd.value.row);
    const minCol = Math.min(selectionStart.value.col, selectionEnd.value.col);
    const maxCol = Math.max(selectionStart.value.col, selectionEnd.value.col);
    for (let row = minRow; row <= maxRow; row++) {
      for (let col = minCol; col <= maxCol; col++) {
        toggleGridSelection({ row, col }, 'select');
      }
    }
    selectionStart.value = null;
    selectionEnd.value = null;
    redrawCanvas();
  }
  lastSelectedCell.value = null;
};

const handleMouseLeave = () => {
  hoveredCell.value = null;
  emit('gridHover', null);
  if (isSelecting.value) {
    isSelecting.value = false;
    lastSelectedCell.value = null;
    if (currentMode.value === 'rectangle') {
      selectionStart.value = null;
      selectionEnd.value = null;
      redrawCanvas();
    }
  }
};

const handleTouchStart = (e: TouchEvent) =>
  handleMouseDown({
    clientX: e.touches[0].clientX,
    clientY: e.touches[0].clientY,
    preventDefault: () => e.preventDefault(),
  } as MouseEvent);
const handleTouchMove = (e: TouchEvent) =>
  handleMouseMove({
    clientX: e.touches[0].clientX,
    clientY: e.touches[0].clientY,
  } as MouseEvent);
const handleTouchEnd = () => handleMouseUp();

// --- Main Logic ---
const getGridCellFromCoordinates = (x: number, y: number): GridCell | null => {
  const rect = canvasRef.value?.getBoundingClientRect();
  if (!rect || !cellSize.value || !gridDimensions.value) return null;

  const canvasX = x - rect.left;
  const canvasY = y - rect.top;

  return screenToGridResolution(
    canvasX,
    canvasY,
    cellSize.value.width,
    cellSize.value.height,
    gridDimensions.value.cols,
    gridDimensions.value.rows
  );
};

const toggleGridSelection = (
  cell: GridCell,
  operation: 'select' | 'deselect' | 'toggle' = 'toggle'
) => {
  if (brushSize.value > 1 && operation !== 'toggle') {
    applyBrush(cell, operation);
  } else {
    toggleSingleCell(cell, operation);
  }
};

const toggleSingleCell = (
  cell: GridCell,
  operation: 'select' | 'deselect' | 'toggle'
) => {
  const key = getGridKey(cell.row, cell.col);
  const activeLayer = colorLayers.value.get(activeColorLayer.value);
  if (!activeLayer) return;

  const isSelected = activeLayer.selectedGrids.has(key);
  switch (operation) {
    case 'select':
      if (!isSelected) activeLayer.selectedGrids.add(key);
      break;
    case 'deselect':
      if (isSelected) activeLayer.selectedGrids.delete(key);
      break;
    case 'toggle':
      if (isSelected) activeLayer.selectedGrids.delete(key);
      else activeLayer.selectedGrids.add(key);
      break;
  }
};

const applyBrush = (centerCell: GridCell, operation: 'select' | 'deselect') => {
  if (!gridDimensions.value) return;

  const { rows, cols } = gridDimensions.value;
  const size = brushSize.value;

  for (let r = centerCell.row - size + 1; r < centerCell.row + size; r++) {
    for (let c = centerCell.col - size + 1; c < centerCell.col + size; c++) {
      if (r >= 0 && r < rows && c >= 0 && c < cols) {
        if (brushShape.value === 'circle') {
          const distance = Math.sqrt(
            Math.pow(r - centerCell.row, 2) + Math.pow(c - centerCell.col, 2)
          );
          if (distance < size) {
            toggleSingleCell({ row: r, col: c }, operation);
          }
        } else {
          // Square brush
          toggleSingleCell({ row: r, col: c }, operation);
        }
      }
    }
  }
};

const updateGridCalculations = () => {
  if (!imageElement.value) return;
  const gridCalc = calculateGridFromResolution(
    imageElement.value.naturalWidth,
    imageElement.value.naturalHeight,
    canvasWidth.value,
    canvasHeight.value,
    props.resolution
  );
  imageAspectRatio.value = gridCalc.aspectRatio;
  gridDimensions.value = gridCalc.gridDimensions;
  cellSize.value = gridCalc.cellSize;
};

const loadImage = async (imageSource: string | File | Blob) => {
  try {
    errorMessage.value = '';
    imageLoaded.value = false;
    const img = new Image();
    img.crossOrigin = 'anonymous';
    await new Promise<void>((resolve, reject) => {
      img.onload = () => {
        imageElement.value = img;
        updateGridCalculations();
        emit('imageLoad', img);
        resolve();
      };
      img.onerror = () => {
        errorMessage.value = '이미지를 불러오는데 실패했습니다.';
        emit('imageError', new Error('Failed to load image'));
        reject(new Error('Failed to load image'));
      };
      if (typeof imageSource === 'string') {
        img.src = imageSource;
      } else {
        const url = URL.createObjectURL(imageSource);
        (img as any)._objectUrl = url;
        img.src = url;
      }
    });
  } catch (error: any) {
    errorMessage.value = error.message;
  } finally {
    imageLoaded.value = true;
    redrawCanvas();
  }
};

// --- Layer Management ---
const setActiveColorLayer = (color: string) => {
  activeColorLayer.value = color.toUpperCase();
  emit('activeLayerChange', activeColorLayer.value);
  redrawCanvas();
};

const toggleLayerVisibility = (color: string) => {
  const layer = colorLayers.value.get(color.toUpperCase());
  if (layer) {
    layer.visible = !layer.visible;
    emit('layerVisibilityChange', color.toUpperCase(), layer.visible);
    redrawCanvas();
  }
};

const getActiveLayerCount = (): number => {
  const activeLayer = colorLayers.value.get(activeColorLayer.value);
  return activeLayer ? activeLayer.selectedGrids.size : 0;
};

const getTotalSelectedCount = (): number => {
  return Array.from(colorLayers.value.values()).reduce(
    (total, layer) => total + (layer.visible ? layer.selectedGrids.size : 0),
    0
  );
};

// --- Brush Methods ---
const setBrushSize = (size: number) => {
  brushSize.value = size;
};

const setBrushShape = (shape: BrushShape) => {
  brushShape.value = shape;
};

// --- Data Export/Import ---
const exportGridLayers = (): ColorLayerExport => {
  const layerData: Record<string, string[]> = {};
  colorLayers.value.forEach((layer: ColorLayer, color: string) => {
    layerData[color] = Array.from(layer.selectedGrids);
  });
  return {
    imageSize: {
      width: imageElement.value?.naturalWidth || 0,
      height: imageElement.value?.naturalHeight || 0,
    },
    resolution: props.resolution,
    layers: layerData,
  };
};

const exportOptimizedLayers = (): GridLayerExport | null => {
  if (!gridDimensions.value) return null;

  const { rows, cols } = gridDimensions.value;
  const layerData: Record<string, Rectangle[]> = {};

  colorLayers.value.forEach((layer: ColorLayer, color: string) => {
    layerData[color] = mergeGridsToRects(layer.selectedGrids, cols, rows);
  });

  return {
    metadata: { cols, rows },
    layers: layerData,
  };
};

const importOptimizedLayers = (data: GridLayerExport) => {
  if (!data || !data.layers || !gridDimensions.value) return;

  const { rows, cols } = gridDimensions.value;

  // For simplicity, we assume the import happens on a grid of the same dimensions.
  // A more advanced version could handle resizing, but that's out of scope for now.
  if (data.metadata.cols !== cols || data.metadata.rows !== rows) {
    console.warn(
      'Importing data from a different grid resolution. Results may be inaccurate.'
    );
  }

  colorLayers.value.clear();

  Object.entries(data.layers).forEach(([color, rects]: [string, any]) => {
    const newSelectedGrids = new Set<string>();
    rects.forEach((rect: any) => {
      for (let r = rect.y; r < rect.y + rect.height; r++) {
        for (let c = rect.x; c < rect.x + rect.width; c++) {
          newSelectedGrids.add(getGridKey(r, c));
        }
      }
    });

    const upperColor = color.toUpperCase();
    colorLayers.value.set(upperColor, {
      color: upperColor,
      selectedGrids: newSelectedGrids,
      visible: true,
      name: getColorName(upperColor),
      opacity: 0.5,
    });
  });

  if (colorLayers.value.size > 0) {
    activeColorLayer.value = Array.from(colorLayers.value.keys())[0];
  }
  redrawCanvas();
  emitSelectionChange();
};

const importGridLayers = (data: ColorLayerExport) => {
  if (!data || !data.layers) return;
  colorLayers.value.clear();
  Object.entries(data.layers).forEach(([color, gridKeys]: [string, any]) => {
    const upperColor = color.toUpperCase();
    colorLayers.value.set(upperColor, {
      color: upperColor,
      selectedGrids: new Set(gridKeys),
      visible: true,
      name: getColorName(upperColor),
      opacity: 0.5,
    });
  });
  if (colorLayers.value.size > 0) {
    activeColorLayer.value = Array.from(colorLayers.value.keys())[0];
  }
  redrawCanvas();
  emitSelectionChange();
};

const emitSelectionChange = () => {
  const layerData: LayerSelectionData = {
    activeColor: activeColorLayer.value,
    layers: Array.from(colorLayers.value.values()).map((l) => ({
      color: l.color,
      selectedCount: l.selectedGrids.size,
      visible: l.visible,
      name: l.name,
    })),
    totalSelected: getTotalSelectedCount(),
  };
  emit('layerChange', layerData);

  const allSelectedCells: GridCell[] = [];
  colorLayers.value.forEach((layer) => {
    if (layer.visible) {
      layer.selectedGrids.forEach((key) =>
        allSelectedCells.push({
          row: parseInt(key.split(',')[0]),
          col: parseInt(key.split(',')[1]),
        })
      );
    }
  });

  const selectionData: ExtendedSelectionData = {
    selectedCells: allSelectedCells,
    resolution: props.resolution,
    imageAspectRatio: imageAspectRatio.value || { width: 1, height: 1 },
    gridDimensions: gridDimensions.value || { cols: 1, rows: 1 },
    cellSize: cellSize.value || { width: 1, height: 1 },
    imageWidth: imageElement.value?.naturalWidth || 0,
    imageHeight: imageElement.value?.naturalHeight || 0,
    canvasWidth: canvasWidth.value,
    canvasHeight: canvasHeight.value,
    layerData: layerData,
  };
  emit('selectionChange', selectionData);
};

// --- Watchers & Lifecycle ---
watch(
  () => props.image,
  (newImage: any) => {
    if (newImage) loadImage(newImage);
  },
  { immediate: true }
);
watch(
  () => props.selectionMode,
  (newMode: any) => {
    if (newMode) switchMode(newMode);
  }
);
watch(
  () => props.resolution,
  () => {
    initColorLayers();
    updateGridCalculations();
    emitSelectionChange();
    redrawCanvas();
  }
);
watch(
  () => props.layerColors,
  () => {
    initColorLayers();
    redrawCanvas();
  },
  { deep: true }
);

defineExpose({
  // Template accessible variables
  canvasWidth,
  canvasHeight,
  imageLoaded,
  errorMessage,
  handleMouseDown,
  handleMouseMove,
  handleMouseUp,
  handleMouseLeave,
  handleTouchStart,
  handleTouchMove,
  handleTouchEnd,
  // Data export/import methods
  exportGridLayers,
  importGridLayers,
  exportOptimizedLayers,
  importOptimizedLayers,
  // External control methods
  switchMode,
  setActiveColorLayer,
  toggleLayerVisibility,
  getColorLayers: () => colorLayers,
  getActiveColorLayer: () => activeColorLayer.value,
  getCurrentMode: () => currentMode.value,
  getBrushSize: () => brushSize.value,
  setBrushSize,
  getBrushShape: () => brushShape.value,
  setBrushShape,
  getTotalSelectedCount,
  getActiveLayerCount,
  getColorName,
});

onMounted(() => {
  initColorLayers();

  // Initialize brush settings from props
  brushSize.value = props.defaultBrushSize;
  brushShape.value = props.defaultBrushShape;

  if (canvasRef.value) {
    redrawCanvas();
  }
});

onUnmounted(() => {
  if (imageElement.value && (imageElement.value as any)._objectUrl) {
    URL.revokeObjectURL((imageElement.value as any)._objectUrl);
  }
});
</script>
<style scoped>
.object-detection-marker {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
  font-family: Arial, sans-serif;
}

.canvas-container {
  position: relative;
  border: 2px solid #dee2e6;
  border-radius: 8px;
  overflow: hidden;
}

.marker-canvas {
  display: block;
  cursor: crosshair;
  background: #ffffff;
}

.loading-indicator,
.error-message {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  padding: 1rem 2rem;
  border-radius: 4px;
  font-weight: 500;
  color: white;
}

.loading-indicator {
  background: rgba(0, 123, 255, 0.9);
}
.error-message {
  background: rgba(220, 53, 69, 0.9);
}
</style>
