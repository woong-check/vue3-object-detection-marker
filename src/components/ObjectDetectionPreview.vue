<template>
  <div class="object-detection-preview">
    <!-- Skeleton Loading -->
    <div v-if="!imageLoaded && !errorMessage" class="skeleton-container">
      <div class="skeleton-shimmer"></div>
      <div class="skeleton-text">미리보기 로딩 중...</div>
    </div>

    <!-- Canvas -->
    <canvas
      ref="previewCanvasRef"
      class="preview-canvas"
      :class="{
        'canvas-loaded': imageLoaded,
        'fit-contain': currentObjectFitMode === 'contain',
        'fit-cover': currentObjectFitMode === 'cover',
      }"
    />

    <!-- Error Message -->
    <div v-if="errorMessage" class="error-message">
      {{ errorMessage }}
    </div>
  </div>
</template>

<script setup lang="ts">
import {
  ref,
  watch,
  onMounted,
  computed,
  defineProps,
  defineExpose,
} from 'vue';
import type { ImageSource, GridLayerExport } from '../types';

// --- Props ---
const props = defineProps<{
  image: ImageSource;
  selectionData: GridLayerExport;
  canvasWidth?: number;
  canvasHeight?: number;
  backgroundColor?: string;
  layerColors?: Record<string, { color: string; opacity: number }>;
  objectFitMode?: 'contain' | 'cover';
  renderMode?: 'grid' | 'rect';
}>();

// --- State ---
const previewCanvasRef = ref<HTMLCanvasElement | null>(null);
const imageLoaded = ref(false);
const errorMessage = ref('');

// --- Computed ---
const currentObjectFitMode = computed(() => props.objectFitMode || 'contain');
const currentRenderMode = computed(() => props.renderMode || 'grid');

// --- Helper Functions ---
const getBoundingRect = (
  rects: Array<{ x: number; y: number; width: number; height: number }>
) => {
  if (rects.length === 0) return null;

  let minX = Infinity;
  let minY = Infinity;
  let maxX = -Infinity;
  let maxY = -Infinity;

  rects.forEach((rect: any) => {
    minX = Math.min(minX, rect.x);
    minY = Math.min(minY, rect.y);
    maxX = Math.max(maxX, rect.x + rect.width);
    maxY = Math.max(maxY, rect.y + rect.height);
  });

  return {
    x: minX,
    y: minY,
    width: maxX - minX,
    height: maxY - minY,
  };
};

// --- Methods ---
const drawPreview = async () => {
  const canvas = previewCanvasRef.value;
  const ctx = canvas?.getContext('2d');
  if (!ctx || !props.image || !canvas) return;

  // Set canvas internal resolution to match its actual display size
  const rect = canvas.getBoundingClientRect();
  canvas.width = rect.width;
  canvas.height = rect.height;

  imageLoaded.value = false;
  errorMessage.value = '';

  try {
    const img = new Image();
    img.crossOrigin = 'anonymous';

    await new Promise<void>((resolve, reject) => {
      img.onload = () => {
        if (!canvas) return;
        ctx.clearRect(0, 0, canvas.width, canvas.height);

        // Calculate image dimensions based on object-fit mode
        const imgRatio = img.width / img.height;
        const canvasRatio = canvas.width / canvas.height;

        let drawWidth, drawHeight, drawX, drawY;

        if (currentObjectFitMode.value === 'contain') {
          // Contain: fit entire image within canvas, maintain aspect ratio
          if (imgRatio > canvasRatio) {
            // Image is wider - fit to width
            drawWidth = canvas.width;
            drawHeight = canvas.width / imgRatio;
            drawX = 0;
            drawY = (canvas.height - drawHeight) / 2;
          } else {
            // Image is taller - fit to height
            drawHeight = canvas.height;
            drawWidth = canvas.height * imgRatio;
            drawX = (canvas.width - drawWidth) / 2;
            drawY = 0;
          }
        } else {
          // Cover: fill entire canvas, maintain aspect ratio, crop if necessary
          if (imgRatio > canvasRatio) {
            // Image is wider - fit to height
            drawHeight = canvas.height;
            drawWidth = canvas.height * imgRatio;
            drawX = (canvas.width - drawWidth) / 2;
            drawY = 0;
          } else {
            // Image is taller - fit to width
            drawWidth = canvas.width;
            drawHeight = canvas.width / imgRatio;
            drawX = 0;
            drawY = (canvas.height - drawHeight) / 2;
          }
        }

        ctx.drawImage(img, drawX, drawY, drawWidth, drawHeight);

        // Draw selection layers
        if (props.selectionData && props.selectionData.layers) {
          const { cols, rows } = props.selectionData.metadata;
          if (cols === 0 || rows === 0) return;

          Object.entries(props.selectionData.layers).forEach(
            ([color, rects]) => {
              if (rects.length === 0) return;

              const layerStyle = props.layerColors?.[color] || {
                color,
                opacity: 0.5,
              };

              if (currentRenderMode.value === 'rect') {
                // Rect mode: draw bounding rectangle for each color
                const boundingRect = getBoundingRect(rects);
                if (!boundingRect) return;

                const x = (boundingRect.x / cols) * canvas.width;
                const y = (boundingRect.y / rows) * canvas.height;
                const width = (boundingRect.width / cols) * canvas.width;
                const height = (boundingRect.height / rows) * canvas.height;

                ctx.fillStyle = hexToRgba(layerStyle.color, layerStyle.opacity);
                ctx.fillRect(x, y, width, height);
              } else {
                // Grid mode: draw individual rectangles (original behavior)
                const path = new Path2D();
                rects.forEach((rect: any) => {
                  const x = (rect.x / cols) * canvas.width;
                  const y = (rect.y / rows) * canvas.height;
                  const width = (rect.width / cols) * canvas.width;
                  const height = (rect.height / rows) * canvas.height;
                  path.rect(x, y, width, height);
                });

                ctx.fillStyle = hexToRgba(layerStyle.color, layerStyle.opacity);
                ctx.fill(path);
              }
            }
          );
        }
        resolve();
      };
      img.onerror = () => reject(new Error('이미지를 불러올 수 없습니다.'));

      if (typeof props.image === 'string') {
        img.src = props.image;
      } else {
        img.src = URL.createObjectURL(props.image);
      }
    });
  } catch (error: any) {
    errorMessage.value = error.message;
  } finally {
    imageLoaded.value = true;
  }
};

const hexToRgba = (hex: string, alpha: number): string => {
  if (!hex || !hex.startsWith('#')) return `rgba(0, 0, 0, ${alpha})`;
  const r = parseInt(hex.slice(1, 3), 16) || 0;
  const g = parseInt(hex.slice(3, 5), 16) || 0;
  const b = parseInt(hex.slice(5, 7), 16) || 0;
  return `rgba(${r}, ${g}, ${b}, ${alpha})`;
};

// --- Watchers & Lifecycle ---
watch(() => [props.image, props.selectionData], drawPreview, { deep: true });

onMounted(() => {
  drawPreview();
});

// Explicitly expose variables to template (workaround for TypeScript)
defineExpose({
  imageLoaded,
  errorMessage,
  currentObjectFitMode,
  currentRenderMode,
});
</script>

<style scoped>
.object-detection-preview {
  position: relative;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #f8f9fa;
  border-radius: 8px;
  overflow: hidden;
}

.preview-canvas {
  max-width: 100%;
  max-height: 100%;
  width: auto;
  height: auto;
  opacity: 0;
  transition: opacity 0.3s ease-in-out;
}

.preview-canvas.fit-contain,
.preview-canvas.fit-cover {
  width: 100%;
  height: 100%;
}

.preview-canvas.canvas-loaded {
  opacity: 1;
}

.skeleton-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background-color: #f8f9fa;
}

.skeleton-shimmer {
  width: 80%;
  height: 60%;
  background: linear-gradient(90deg, #e2e8f0 25%, #f1f5f9 50%, #e2e8f0 75%);
  background-size: 200% 100%;
  animation: shimmer 2s infinite;
  border-radius: 8px;
  margin-bottom: 1rem;
}

@keyframes shimmer {
  0% {
    background-position: -200% 0;
  }
  100% {
    background-position: 200% 0;
  }
}

.skeleton-text {
  color: #64748b;
  font-size: 0.9rem;
  font-weight: 500;
  text-align: center;
  padding: 0.5rem;
  background-color: rgba(255, 255, 255, 0.8);
  border-radius: 4px;
  backdrop-filter: blur(4px);
}

.error-message {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  padding: 1rem;
  border-radius: 8px;
  background-color: rgba(239, 68, 68, 0.9);
  color: white;
  font-weight: bold;
  text-align: center;
  backdrop-filter: blur(4px);
}
</style>
