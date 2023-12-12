<script setup lang="ts">
const files = ref<FileList | null>(null);
const filesLength = computed(() => (files.value ? files.value.length : 0));
const pageWidth = 210 - 12 * 2; // A4, in mm
const pageHeight = 297 - 12 * 2;
const gap = 10;
const maxWidth = (pageWidth - 3 * gap) / 2;
const maxHeight = (pageHeight - 3 * gap) / 2;
const maxRatio = maxWidth / maxHeight;

async function readFile(file: File): Promise<string> {
  return new Promise((resolve, reject) => {
    var fr = new FileReader();
    fr.onload = () => {
      resolve(fr.result as string);
    };
    fr.onerror = reject;
    fr.readAsDataURL(file);
  });
}
const showInput = computed(() => !files.value);

class ImageItem {
  id: number;
  ref?: HTMLImageElement;
  dataUrl: string;
  initialSize?: { width: number; height: number };
  size?: { width: number; height: number };

  getRatio() {
    if (!this.initialSize) throw new Error("initialSize is not set");
    return this.initialSize.width / this.initialSize.height;
  }
  getScale() {
    if (!this.initialSize) throw new Error("initialSize is not set");
    return Math.min(
      maxWidth / this.initialSize.width,
      maxHeight / this.initialSize.height
    );
  }
  scale() {
    if (!this.initialSize) throw new Error("initialSize is not set");
    const scale = this.getScale();
    this.size = {
      width: this.initialSize.width * scale,
      height: this.initialSize.height * scale,
    };
  }

  constructor(id: number, dataUrl: string) {
    this.id = id;
    this.dataUrl = dataUrl;
    this.size = undefined;
  }
}

function onFileSelected(e: Event) {
  const target = e.target as HTMLInputElement;
  files.value = target.files;
}

const imageItems = shallowRef<ImageItem[][]>([]);
watch(files, async (files) => {
  if (!files) return;
  const flatImageItems = await Promise.all(
    Array.from(files).map(async (file, id) => {
      const dataUrl = await readFile(file);
      return new ImageItem(id, dataUrl);
    })
  );
  for (let i = 0; i < flatImageItems.length; i += 4) {
    imageItems.value.push(flatImageItems.slice(i, i + 4));
  }
  triggerRef(imageItems);
});

function getOnload(image: ImageItem) {
  function onload(event: Event) {
    const target = event.target as HTMLImageElement;
    image.initialSize = {
      width: target.naturalWidth,
      height: target.naturalHeight,
    };
    image.scale();
    if (image.id === filesLength.value - 1) {
      triggerRef(imageItems);
    }
  }
  return onload;
}

function getImageStyle(image: ImageItem) {
  return {
    width: `${image.size?.width}mm`,
    height: `${image.size?.height}mm`,
    marginRight: image.id % 2 === 0 ? `${gap}mm` : `0`,
    marginBottom: image.id % 4 < 2 ? `${gap}mm` : `0`,
  } satisfies Partial<CSSStyleDeclaration>;
}
</script>

<template>
  <input
    v-if="showInput"
    type="file"
    multiple
    @change="onFileSelected"
    accept="image/*"
  />
  <div v-else>
    <div
      class="quad-image-container"
      v-for="quadImage in imageItems"
      :style="{
        height: `${pageHeight}mm`,
      }"
    >
      <img
        v-for="image in quadImage"
        :style="getImageStyle(image)"
        :src="image.dataUrl"
        @load="(event) => getOnload(image)(event)"
      />
    </div>
  </div>
</template>

<style>
.quad-image-container {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  grid-template-rows: repeat(2, 1fr);
  page-break-after: always;
  page-break-inside: avoid;
  overflow: hidden;
}

.quad-image-container img {
  align-self: center;
}

@page {
  size: A4;
  margin: 10mm;
}
</style>
