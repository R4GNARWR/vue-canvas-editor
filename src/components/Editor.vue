<template>
    <div class="canvas-wrap">
        <canvas id="editor" ref="mainCanvas" class="main-canvas"></canvas>
        <div class="canvas-actions">
            <button @click="addImageInput?.click()">
                Добавить изображение
                <input type="file" ref="addImageInput" accept="image/png, image/jpg, image/jpeg"
                    @change="addImageToCanvas($event)">
            </button>
            <button @click="addTextToCanvas">Добавить текст</button>
        </div>
        <div ref="imageBox" class="image-box">
            <img ref="mainImage" :src="baseImageOptions.src" v-if="baseImageOptions.src" />
        </div>
    </div>
</template>

<script setup lang="ts">
import { fabric } from 'fabric';
import { onMounted, Ref, ref, watch, PropType } from 'vue';

type ClipPathOptions = {
    width: number,
    height: number,
    x: number,
    y: number
}
type BaseImageOptions = {
    src: string,
    x: number,
    y: number,
    width: number,
    height: number
}
const props = defineProps({
    textColor: {
        type: String,
        default: '#000000'
    },
    clipPathOptions: {
        type: Object as PropType<ClipPathOptions>,
        default: () => ({})
    },
    baseImageOptions: {
        type: Object as PropType<BaseImageOptions>,
        default: () => ({})
    },
})
const imageBox: Ref<HTMLDivElement | null> = ref(null);
const mainImage: Ref<HTMLImageElement | null> = ref(null);
const mainCanvas: Ref<HTMLCanvasElement | null> = ref(null);
const addImageInput: Ref<HTMLInputElement | null> = ref(null);
const clipPath: Ref<fabric.Rect | null> = ref(null);
const canvasInstance: Ref<fabric.Canvas | null> = ref(null);
const mainImgElement: Ref<fabric.Image | null> = ref(null);
const imagesElements: Ref<fabric.Image[]> = ref([]);
const textElements: Ref<fabric.Textbox[]> = ref([]);
const selectedElement: Ref<fabric.Object | null | undefined> = ref(null);

const disableInteractions = (element: fabric.Object) => {
    if (!element) return;
    element.lockMovementX = true;
    element.lockMovementY = true;
    element.selectable = false;
}
const initializeFabric = async () => {
    const options = {
        width: 700,
        height: window.innerHeight - 100,
        backgroundColor: '#FFFFFF',
        preserveObjectStacking: true
    };
    if (mainCanvas.value) {
        canvasInstance.value = new fabric.Canvas(mainCanvas.value, options);
        canvasInstance.value.on('selection:cleared', () => {
            selectedElement.value = null;
        })
        canvasInstance.value.on('selection:updated', () => {
            const selectedObject = canvasInstance.value?.getActiveObject();
            if (selectedObject) selectedElement.value = selectedObject;
        })
    }
    if (mainImage.value) {
        mainImage.value.onload = () => {
            if (mainImage.value) {
                mainImgElement.value = new fabric.Image(mainImage.value, {
                    left: props.baseImageOptions.x || 0,
                    top: props.baseImageOptions.y || 100,
                    hoverCursor: 'auto',
                });
                mainImgElement.value.scaleToHeight(props.baseImageOptions.height || 500);
                mainImgElement.value.scaleToWidth(props.baseImageOptions.width || 750);
                disableInteractions(mainImgElement.value);
                if (canvasInstance.value) {
                    canvasInstance.value.add(mainImgElement.value);
                    createClippingArea(mainImgElement.value);
                }
            }
        }
    }
}

const createClippingArea = (element: fabric.Image) => {
    if (!element) return;
    const width = element.getScaledWidth() * 0.44;
    const height = element.getScaledHeight() * 0.46;

    const clipRect = new fabric.Rect({
        width: props.clipPathOptions.width || width,
        height: props.clipPathOptions.height || height,
        left: props.clipPathOptions.x || element.getCenterPoint().x - (width / 2) + 3.5,
        top: props.clipPathOptions.y || element.getCenterPoint().y - (height / 2) - 51.5,
        absolutePositioned: true,
    })

    clipPath.value = clipRect;
}

const addTextToCanvas = () => {
    const newText = new fabric.Textbox('Введите текст', {
        left: mainImgElement.value?.getCenterPoint().x || 0,
        top: mainImgElement.value?.getCenterPoint().y || 0,
        borderColor: '#000000',
        cornerColor: '#000000',
        fill: props.textColor,
    })
    if (clipPath.value) {
        newText.clipPath = clipPath.value;
    }
    textElements.value?.push(newText);
    if (canvasInstance.value) {
        canvasInstance.value.add(textElements.value[textElements.value.length - 1]);
        textElements.value[textElements.value.length - 1].bringForward().bringForward();
        textElements.value[textElements.value.length - 1].on('selected', () => {
            selectedElement.value = textElements.value[textElements.value.length - 1];
        });
    }
}
const changeTextColor = (color: string) => {
    if (selectedElement.value && selectedElement.value instanceof fabric.Textbox) {
        selectedElement.value.set('fill', color);
        canvasInstance.value?.renderAll();
    };
}

const addImageToCanvas = (event: Event) => {
    if (!canvasInstance.value) return;
    const file = (event.target as HTMLInputElement).files?.[0];
    if (!file) return;
    const imageEL = document.createElement('img');
    imageEL.src = URL.createObjectURL(file);
    if (imageBox.value) {
        imageBox.value.appendChild(imageEL);
    }

    imageEL.onload = () => {
        const newImage = new fabric.Image(imageEL, {
            evented: true,
            left: 0,
            top: 100,
            borderColor: '#000000',
            cornerColor: '#000000',
        });
        if (imageEL && clipPath.value) {
            // Получаем размеры контейнера
            const containerWidth = clipPath.value?.getScaledWidth();
            const containerHeight = clipPath.value?.getScaledHeight();

            if (containerWidth && containerHeight) {
                // Вычисляем коэффициенты масштабирования
                const widthRatio = containerWidth / imageEL.clientWidth;
                const heightRatio = containerHeight / imageEL.clientHeight;
                const scale = Math.min(widthRatio, heightRatio);
                if (scale) {
                    // Масштабируем изображение
                    newImage.scaleToHeight(imageEL.clientHeight * scale);
                    newImage.scaleToWidth(imageEL.clientWidth * scale);
                }

            }
        }

        if (clipPath.value) {
            newImage.clipPath = clipPath.value;
        }

        if (mainImgElement.value) {
            newImage.left = mainImgElement.value.getCenterPoint().x - (newImage.getScaledWidth() / 2);
            newImage.top = mainImgElement.value.getCenterPoint().y - (newImage.getScaledHeight() / 2);
        } else {
            newImage.left = 0;
            newImage.top = 0;
        }

        imagesElements.value?.push(newImage);
        canvasInstance.value?.add(imagesElements.value[imagesElements.value.length - 1]);
        imagesElements.value[imagesElements.value.length - 1].bringForward();
        imagesElements.value[imagesElements.value.length - 1].on('selected', () => {
            selectedElement.value = imagesElements.value[imagesElements.value.length - 1];
        });
    }
}

const deleteElement = () => {
    if (selectedElement.value && canvasInstance.value) {
        if (selectedElement.value.type === 'image') {
            const deletedElementIndex = imagesElements.value?.findIndex((el) => {
                if (!el.ownMatrixCache || !selectedElement.value.ownMatrixCache) return false
                if (el.ownMatrixCache?.key === selectedElement.value.ownMatrixCache?.key) return true
                else return false
            });
            const deletedElement = imagesElements.value[deletedElementIndex];
            console.log(deletedElement)
            if (deletedElementIndex !== -1) {
                canvasInstance.value.remove(imagesElements.value[deletedElementIndex]);
                imagesElements.value.splice(deletedElementIndex, 1);
            }
        }
        else if (selectedElement.value.type === 'textbox') {
            const deletedElementIndex = textElements.value?.findIndex((el) => el.selected === true);
            const deletedElement = textElements.value[deletedElementIndex];
            if (deletedElementIndex !== -1) {
                canvasInstance.value.remove(textElements.value[deletedElementIndex]);
                textElements.value.splice(deletedElementIndex, 1);
            }
        }
        selectedElement.value = null;
    }

}

onMounted(() => {
    window.addEventListener('keydown', (event) => {
        if (event.code === 'Delete') deleteElement();
    })
    initializeFabric();
})

watch(() => props.textColor, (newColor) => {
    changeTextColor(newColor)
})
</script>

<style lang="scss">
.canvas-wrap {
    position: relative;
    width: fit-content;
    overflow: hidden;
}

.canvas-actions {
    position: absolute;
    bottom: 10px;
    left: 0;
    width: 100%;
    display: flex;
    justify-content: center;
    column-gap: 10px;

    button {
        position: relative;
        padding: 7px 20px;
        font-size: 14px;
        line-height: 1em;
        border-radius: 10px;
        background-color: #006c80;
        outline: none;
        border: 1px solid #006c80;
        color: #FFFFFF;
        cursor: pointer;

        input {
            position: absolute;
            user-select: none;
            opacity: 0.01;
        }
    }
}

.image-box {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: -1;
    opacity: 0;
}
</style>