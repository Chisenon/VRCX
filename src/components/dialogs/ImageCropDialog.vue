<template>
    <Dialog
        :open="open"
        @update:open="
            (v) => {
                if (!v) cancelCrop();
            }
        ">
        <DialogContent class="x-dialog sm:max-w-212.5">
            <DialogHeader>
                <DialogTitle>{{ title }}</DialogTitle>
            </DialogHeader>

            <div v-if="cropperImageSrc" class="mt-4">
                <Cropper
                    ref="cropperRef"
                    class="h-100 max-h-full"
                    :src="cropperImageSrc"
                    :stencil-props="{ aspectRatio, movable: !loading, resizable: !loading }"
                    :move-image="!loading"
                    :resize-image="!loading"
                    :image-restriction="freeMode ? 'none' : 'stencil'"
                    @ready="handleCropperReady"
                    @change="onCropperChange" />

                <!-- Toolbar -->
                <div class="flex items-center justify-center gap-1 mt-3">
                    <TooltipWrapper :content="t('dialog.image_crop.rotate_left')">
                        <Button
                            size="icon-sm"
                            variant="outline"
                            class="rounded-full h-8 w-8"
                            :disabled="loading"
                            @click="cropperRef?.rotate(-90)">
                            <RotateCcw class="h-4 w-4" />
                        </Button>
                    </TooltipWrapper>
                    <TooltipWrapper :content="t('dialog.image_crop.rotate_right')">
                        <Button
                            size="icon-sm"
                            variant="outline"
                            class="rounded-full h-8 w-8"
                            :disabled="loading"
                            @click="cropperRef?.rotate(90)">
                            <RotateCw class="h-4 w-4" />
                        </Button>
                    </TooltipWrapper>

                    <div class="w-px h-5 bg-border mx-1" />

                    <TooltipWrapper :content="t('dialog.image_crop.flip_h')">
                        <Button
                            size="icon-sm"
                            variant="outline"
                            class="rounded-full h-8 w-8"
                            :disabled="loading"
                            @click="cropperRef?.flip(true, false)">
                            <FlipHorizontal class="h-4 w-4" />
                        </Button>
                    </TooltipWrapper>
                    <TooltipWrapper :content="t('dialog.image_crop.flip_v')">
                        <Button
                            size="icon-sm"
                            variant="outline"
                            class="rounded-full h-8 w-8"
                            :disabled="loading"
                            @click="cropperRef?.flip(false, true)">
                            <FlipVertical class="h-4 w-4" />
                        </Button>
                    </TooltipWrapper>

                    <div class="w-px h-5 bg-border mx-1" />

                    <TooltipWrapper :content="t('dialog.image_crop.zoom_out')">
                        <Button
                            size="icon-sm"
                            variant="ghost"
                            class="rounded-full h-7 w-7"
                            :disabled="loading"
                            @click="cropperRef?.zoom(0.8)">
                            <ZoomOut class="h-3.5 w-3.5" />
                        </Button>
                    </TooltipWrapper>
                    <Slider
                        v-model="zoomSliderValue"
                        :min="0"
                        :max="100"
                        :step="1"
                        :disabled="loading"
                        class="w-28"
                        @value-commit="onZoomCommit" />
                    <TooltipWrapper :content="t('dialog.image_crop.zoom_in')">
                        <Button
                            size="icon-sm"
                            variant="ghost"
                            class="rounded-full h-7 w-7"
                            :disabled="loading"
                            @click="cropperRef?.zoom(1.2)">
                            <ZoomIn class="h-3.5 w-3.5" />
                        </Button>
                    </TooltipWrapper>

                    <div class="w-px h-5 bg-border mx-1" />

                    <TooltipWrapper
                        :content="freeMode ? t('dialog.image_crop.mode_fit') : t('dialog.image_crop.mode_free')">
                        <Button
                            data-testid="crop-mode-toggle"
                            size="icon-sm"
                            :variant="freeMode ? 'default' : 'outline'"
                            class="rounded-full h-8 w-8"
                            :disabled="loading"
                            @click="toggleMode">
                            <Expand v-if="freeMode" class="h-4 w-4" />
                            <Frame v-else class="h-4 w-4" />
                        </Button>
                    </TooltipWrapper>

                    <div class="w-px h-5 bg-border mx-1" />

                    <TooltipWrapper :content="fillTooltip">
                        <Button
                            data-testid="fill-mode-toggle"
                            size="icon-sm"
                            variant="outline"
                            class="rounded-full h-8 w-8"
                            :disabled="loading"
                            @click="toggleFillMode">
                            <ArrowUpDown v-if="fillMode === 'vertical'" class="h-4 w-4" />
                            <ArrowLeftRight v-else class="h-4 w-4" />
                        </Button>
                    </TooltipWrapper>

                    <TooltipWrapper :content="t('dialog.image_crop.reset')">
                        <Button
                            size="icon-sm"
                            variant="outline"
                            class="rounded-full h-8 w-8"
                            :disabled="loading"
                            @click="handleReset">
                            <RefreshCw class="h-4 w-4" />
                        </Button>
                    </TooltipWrapper>
                </div>
            </div>

            <DialogFooter>
                <template v-if="cropperImageSrc">
                    <Button variant="secondary" size="sm" :disabled="loading" @click="cancelCrop">
                        {{ t('dialog.change_content_image.cancel') }}
                    </Button>
                    <Button size="sm" :disabled="loading" @click="onConfirmCrop">
                        <Spinner v-if="loading" />
                        {{ loading ? t('message.upload.loading') : t('dialog.gallery_icons.crop_image') }}
                    </Button>
                </template>
            </DialogFooter>
        </DialogContent>
    </Dialog>
</template>

<script setup>
    import {
        ArrowLeftRight,
        ArrowUpDown,
        Expand,
        FlipHorizontal,
        FlipVertical,
        Frame,
        RefreshCw,
        RotateCcw,
        RotateCw,
        ZoomIn,
        ZoomOut
    } from 'lucide-vue-next';
    import { Dialog, DialogContent, DialogFooter, DialogHeader, DialogTitle } from '@/components/ui/dialog';
    import { computed, nextTick, ref, watch } from 'vue';
    import { Button } from '@/components/ui/button';
    import { Cropper } from 'vue-advanced-cropper';
    import { Slider } from '@/components/ui/slider';
    import { Spinner } from '@/components/ui/spinner';
    import { useI18n } from 'vue-i18n';

    import TooltipWrapper from '@/components/ui/tooltip/TooltipWrapper.vue';

    import { useImageCropper } from '../../composables/useImageCropper';

    import 'vue-advanced-cropper/dist/style.css';

    const { t } = useI18n();

    const props = defineProps({
        open: {
            type: Boolean,
            required: true
        },
        title: {
            type: String,
            default: ''
        },
        aspectRatio: {
            type: Number,
            default: 4 / 3
        },
        file: {
            type: [File, null],
            default: null
        }
    });

    const emit = defineEmits(['update:open', 'confirm']);

    const loading = ref(false);
    const freeMode = ref(false);
    const fitCropperToken = ref(0);
    const fillMode = ref('vertical');
    const VISIBLE_AREA_MARGIN_PX = 10;
    const AUTO_FIT_MAX_STEPS = 4;
    const AUTO_FIT_EPSILON = 0.001;

    const fillTooltip = computed(() => {
        return t(`dialog.image_crop.fill_${fillMode.value}`);
    });

    const zoomSliderValue = ref([50]);
    const lastZoomRatio = ref(1);

    const MIN_ZOOM_RATIO = 0.3;
    const MAX_ZOOM_RATIO = 5;
    const LOG_MIN = Math.log(MIN_ZOOM_RATIO);
    const LOG_MAX = Math.log(MAX_ZOOM_RATIO);

    const { cropperRef, cropperImageSrc, resetCropState, loadImageForCrop, getCroppedBlob } = useImageCropper();

    watch(
        () => props.file,
        (file) => {
            if (file) {
                loadImageForCrop(file);
            }
        }
    );

    watch(
        () => props.open,
        (open) => {
            if (!open) {
                loading.value = false;
                freeMode.value = false;
                fillMode.value = 'vertical';
                zoomSliderValue.value = [50];
                lastZoomRatio.value = 1;
                resetCropState();
            }
        }
    );

    watch(
        () => freeMode.value,
        (isFree, wasFree) => {
            if (!isFree && wasFree) {
                scheduleFitCropper();
            }
        },
        { flush: 'post' }
    );

    /**
     * @param result
     */
    function onCropperChange(result) {
        if (!result.visibleArea || !result.image) return;
        const ratio = result.image.width / result.visibleArea.width;
        lastZoomRatio.value = ratio;
        const normalized = ((Math.log(ratio) - LOG_MIN) / (LOG_MAX - LOG_MIN)) * 100;
        zoomSliderValue.value = [Math.max(0, Math.min(100, Math.round(normalized)))];
    }

    /**
     * @param value
     */
    function onZoomCommit(value) {
        const target = value[0];
        const targetRatio = Math.exp(LOG_MIN + (target / 100) * (LOG_MAX - LOG_MIN));
        const factor = targetRatio / lastZoomRatio.value;
        cropperRef.value?.zoom(factor);
    }

    /**
     *
     */
    async function applyFillMode(mode) {
        if (!cropperRef.value) return;
        await ensureImageFitsEditableArea();

        cropperRef.value.setCoordinates(({ imageSize, coordinates }) => {
            if (!imageSize?.width || !imageSize?.height) {
                return coordinates || {};
            }
            const stencilAspect =
                coordinates?.width && coordinates?.height
                    ? coordinates.width / coordinates.height
                    : props.aspectRatio;
            return calculateCenteredFillCoordinates(
                mode,
                imageSize.width,
                imageSize.height,
                stencilAspect
            );
        }, {
            transitions: true,
            autoZoom: false
        });
        await nextTick();
        await ensureStencilFitsEditableArea();
    }

    /**
     *
     */
    async function toggleFillMode() {
        await applyFillMode(fillMode.value);
        fillMode.value = fillMode.value === 'vertical' ? 'horizontal' : 'vertical';
    }

    /**
     *
     */
    function fillCropper() {
        cropperRef.value?.setCoordinates(
            ({ imageSize }) => ({
                left: 0,
                top: 0,
                width: imageSize.width,
                height: imageSize.height
            }),
            {
                transitions: false,
                autoZoom: false
            }
        );
    }

    /**
     * @param {'vertical' | 'horizontal'} mode
     * @param {number} imageWidth
     * @param {number} imageHeight
     * @param {number} stencilAspect
     */
    function calculateCenteredFillCoordinates(
        mode,
        imageWidth,
        imageHeight,
        stencilAspect
    ) {
        let width;
        let height;
        if (mode === 'vertical') {
            height = imageHeight;
            width = height * stencilAspect;
        } else {
            width = imageWidth;
            height = width / stencilAspect;
        }
        return {
            left: (imageWidth - width) / 2,
            top: (imageHeight - height) / 2,
            width,
            height
        };
    }

    /**
     * Keep the stencil inside the visible editor area by zooming out when needed.
     * @param {any} [result]
     */
    async function ensureImageFitsEditableArea() {
        if (!cropperRef.value) return;
        for (let i = 0; i < AUTO_FIT_MAX_STEPS; i += 1) {
            const result = cropperRef.value.getResult();
            if (!result?.visibleArea || !result?.image) return;
            const { x: marginX, y: marginY } = getVisibleAreaMarginInImageUnits(result);
            const imageOverflow = getImageOverflowFromVisibleArea(result, marginX, marginY);
            if (imageOverflow.max <= AUTO_FIT_EPSILON) {
                return;
            }

            const targetVisibleWidth = result.image.width + marginX * 2;
            const targetVisibleHeight = result.image.height + marginY * 2;
            const ratio = Math.max(
                targetVisibleWidth / Math.max(1, result.visibleArea.width),
                targetVisibleHeight / Math.max(1, result.visibleArea.height)
            );
            if (ratio <= 1 + AUTO_FIT_EPSILON) {
                return;
            }

            const center = {
                left: result.image.width / 2,
                top: result.image.height / 2
            };
            cropperRef.value.zoom(Math.max(0.1, (1 / ratio) * 0.995), center);
            await nextTick();
        }
    }

    /**
     *
     */
    async function ensureStencilFitsEditableArea() {
        if (!cropperRef.value) return;
        for (let i = 0; i < AUTO_FIT_MAX_STEPS; i += 1) {
            const result = cropperRef.value.getResult();
            if (!result?.coordinates || !result?.visibleArea) return;
            const { x: marginX, y: marginY } = getVisibleAreaMarginInImageUnits(result);
            const maxStencilWidth = Math.max(1, result.visibleArea.width - marginX * 2);
            const maxStencilHeight = Math.max(1, result.visibleArea.height - marginY * 2);
            const overflowRatio = Math.max(
                result.coordinates.width / maxStencilWidth,
                result.coordinates.height / maxStencilHeight
            );
            if (overflowRatio <= 1 + AUTO_FIT_EPSILON) {
                return;
            }

            const center = {
                left: result.visibleArea.left + result.visibleArea.width / 2,
                top: result.visibleArea.top + result.visibleArea.height / 2
            };
            cropperRef.value.zoom(Math.max(0.1, (1 / overflowRatio) * 0.995), center);
            await nextTick();
        }
    }

    /**
     * Detect whether editable area exceeds image bounds.
     * @param {any} result
     * @param {number} marginX
     * @param {number} marginY
     */
    function getImageOverflowFromVisibleArea(result, marginX, marginY) {
        const imageWidth = result.image?.width || 0;
        const imageHeight = result.image?.height || 0;
        const left = result.visibleArea.left || 0;
        const top = result.visibleArea.top || 0;
        const right = left + (result.visibleArea.width || 0);
        const bottom = top + (result.visibleArea.height || 0);

        const overflowLeft = Math.max(0, marginX - left);
        const overflowTop = Math.max(0, marginY - top);
        const overflowRight = Math.max(0, right - (imageWidth - marginX));
        const overflowBottom = Math.max(0, bottom - (imageHeight - marginY));

        return {
            left: overflowLeft,
            top: overflowTop,
            right: overflowRight,
            bottom: overflowBottom,
            max: Math.max(overflowLeft, overflowTop, overflowRight, overflowBottom)
        };
    }

    /**
     * Convert UI px margin into cropper image-coordinate units.
     * @param {any} result
     */
    function getVisibleAreaMarginInImageUnits(result) {
        const root = cropperRef.value?.$el;
        const boundaries = root?.querySelector?.('.vue-advanced-cropper__boundaries');
        const target = boundaries || root;
        const rect = target?.getBoundingClientRect?.();
        if (!rect?.width || !rect?.height) {
            return {
                x: VISIBLE_AREA_MARGIN_PX,
                y: VISIBLE_AREA_MARGIN_PX
            };
        }
        return {
            x: (result.visibleArea.width / rect.width) * VISIBLE_AREA_MARGIN_PX,
            y: (result.visibleArea.height / rect.height) * VISIBLE_AREA_MARGIN_PX
        };
    }

    /**
     *
     */
    async function scheduleFitCropper() {
        const token = ++fitCropperToken.value;
        await nextTick();
        await nextTick();

        if (fitCropperToken.value !== token || freeMode.value) {
            return;
        }

        fillCropper();
    }

    /**
     *
     */
    function handleCropperReady() {
        if (!freeMode.value) {
            scheduleFitCropper();
        }
    }

    /**
     *
     */
    function toggleMode() {
        freeMode.value = !freeMode.value;
    }

    /**
     *
     */
    function handleReset() {
        freeMode.value = false;
        fillMode.value = 'vertical';
        scheduleFitCropper();
    }

    /**
     *
     */
    function cancelCrop() {
        resetCropState();
        emit('update:open', false);
    }

    /**
     *
     */
    async function onConfirmCrop() {
        loading.value = true;
        try {
            const blob = await getCroppedBlob(props.file);
            if (!blob) {
                loading.value = false;
                return;
            }
            emit('confirm', blob);
        } catch {
            loading.value = false;
        }
    }
</script>
