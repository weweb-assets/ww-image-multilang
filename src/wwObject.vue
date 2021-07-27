<template>
    <div class="ww-image" :class="{ '-can-drag-image': isDoubleSelected && !isMoving }" v-on="dragListener">
        <!-- wwEditor:start -->
        <div class="ww-image__drag-overlay">
            <wwEditorIcon class="ww-image__drag-overlay-icon" name="move" />
        </div>
        <!-- wwEditor:end -->
        <div class="ww-image__wrapper" :style="formatStyle" ww-responsive="ww-img-wrap">
            <div class="ww-image__ratio" :style="ratioStyle" ww-responsive="ww-img-ratio"></div>
            <!-- wwEditor:start -->
            <img
                draggable="false"
                class="ww-image__img"
                :src="source"
                :alt="wwLang.getText(content.alt)"
                :style="imageStyle"
                ww-responsive="ww-img"
            />
            <!-- wwEditor:end -->

            <!-- wwFront:start -->
            <!-- NO SRCSET -->
            <img
                v-if="loadWithTwicPics"
                class="ww-image__img twic"
                :src="placeholder"
                :data-twic-src="twicPicsDataSrc"
                data-src-transform="quality=85/auto"
                data-src-step="10"
                :alt="wwLang.getText(content.alt)"
                :style="imageStyle"
                ww-responsive="ww-img-twic"
            />

            <!-- SRCSET -->
            <picture v-else class="ww-image__img-picture" loading="lazy" ww-responsive="ww-img">
                <source v-for="(srcset, index) in imgSrcSet" :key="index" :srcset="srcset.src" :media="srcset.media" />
                <img
                    class="ww-image__img"
                    :style="imageStyle"
                    :src="source"
                    :alt="wwLang.getText(content.alt)"
                    loading="lazy"
                />
            </picture>
            <!-- wwFront:end -->
        </div>
    </div>
</template>

<script>
export default {
    wwDefaultContent: {
        alt: { en: '' },
        url: wwLib.responsive('https://cdn.weweb.app/public/images/no_image_selected.png'),
        x: wwLib.allowState(wwLib.responsive(0)),
        y: wwLib.allowState(wwLib.responsive(0)),
        zoom: wwLib.allowState(wwLib.responsive(1)),
        style: wwLib.allowState({}),
        focusPoint: wwLib.responsive([50, 50]),
        ratio: wwLib.responsive(0),
    },
    inject: {
        getObjectStyle: { default: () => {} },
    },
    props: {
        content: { type: Object, required: true },
        wwElementState: { type: Object, required: true },
        wwFrontState: { type: Object, required: true },
        bonjour: String,
        /* wwManager:start */
        wwEditorState: { type: Object, required: true },
        /* wwManager:end */
    },
    emits: ['update:content'],
    data() {
        return {
            placeholder: 'data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==',
            imgSrcSet: [],
            wwLang: wwLib.wwLang,
            dragListener: {},

            /* wwManager:start */
            lastMovePosition: { x: 0, y: 0 },
            initialPosition: { x: 0, y: 0 },
            lastTouchDist: 0,
            isMoving: false,
            zoomBarElement: null,
            moveDirection: null,
            /* wwManager:end */
        };
    },
    computed: {
        isPrerender() {
            return window.__WW_IS_PRERENDER__;
        },
        url() {
            return this.wwElementState.props.url || this.content.url;
        },
        loadWithTwicPics() {
            return !this.isPrerender && !this.imgSrcSet && !this.url.startsWith('http');
        },
        twicPicsDataSrc() {
            return `image:${wwLib.wwUtils.getTwicPicsFolder()}${this.url}`;
        },
        screenSize() {
            return this.$store.getters['front/getScreenSize'];
        },
        screenSizes() {
            return this.$store.getters['front/getScreenSizes'];
        },
        imageStyle() {
            let style = {
                filter:
                    this.content && this.content.style && this.content.style.filter ? this.content.style.filter : null,
                '--zoom': (this.content && this.content.zoom) || 1,
                '--left': (this.content && this.content.x) || 0,
                '--top': (this.content && this.content.y) || 0,
                transition: this.getObjectStyle().transition,
            };
            return style;
        },
        formatStyle() {
            const overlayBackground = (this.content.style && this.content.style.overlay) || 'none';
            return {
                '--ww-image-overlay-background': overlayBackground,
            };
        },
        ratioStyle() {
            if (!this.getObjectStyle().height || this.getObjectStyle().height === 'auto') {
                return {
                    '--ww-image-ratio': `${this.content.ratio * 100}%`,
                };
            } else {
                return {
                    '--ww-image-ratio': '0%',
                };
            }
        },
        focusPoint() {
            return `${this.content.focusPoint[0]}px${this.content.focusPoint[1]}p`;
        },
        isDoubleSelected() {
            /* wwEditor:start */
            return this.wwEditorState.isDoubleSelected;
            /* wwEditor:end */
            // eslint-disable-next-line no-unreachable
            return false;
        },
        source() {
            if (!this.url.startsWith('http')) {
                return `${wwLib.wwUtils.getCdnPrefix()}${this.url}`;
            } else {
                return this.url;
            }
        },
    },
    watch: {
        /* wwEditor:start */
        // 'content.url': {
        //     handler(newUrl, oldUrl) {
        //         if (newUrl === oldUrl || !this.$el) return;

        //         const img = this.$el.querySelector('img');
        //         if (!img) return;

        //         img.onload = () => {
        //             if (!img.naturalWidth || !img.naturalHeight) return;
        //             const ratio = img.naturalHeight / img.naturalWidth;
        //             this.$emit('update:content', { ratio });
        //         };
        //     },
        // },
        isDoubleSelected() {
            if (this.isDoubleSelected) {
                this.dragListener = {
                    mousedown: this.startMove,
                };
            } else {
                this.dragListener = {};
            }
        },
        /* wwEditor:end */
        /* wwFront:start */
        screenSize(newValue, oldValue) {
            if (this.isPrerender && oldValue != newValue) {
                this.setSrcSet();
            }
        },
        /* wwFront:end */
    },
    created() {
        /* wwFront:start */
        const splited = this.wwElementState.uid.split('-');
        const troncatedUid = splited[splited.length - 1];
        if (window[`wwg_imgsrcset_${troncatedUid}`]) {
            this.imgSrcSet = window[`wwg_imgsrcset_${troncatedUid}`];
        }

        /* wwFront:end */
    },
    mounted() {
        if (this.isPrerender) {
            this.setSrcSet();

            this.$el.setAttribute('bonjour', 'oui salut');
        }

        /* wwManager:start */
        // if (!this.$el) return;

        // const img = this.$el.querySelector('img');
        // if (!img) return;

        // img.onload = () => {
        //     if (!img.naturalWidth || !img.naturalHeight) return;
        //     const ratio = img.naturalHeight / img.naturalWidth;
        //     this.$emit('update:content', { ratio });
        // };
        /* wwManager:end */
    },
    methods: {
        setSrcSet() {
            //document.querySelector('[data-ww-uid="291415a8-1cd3-428c-9c29-34c6773ddb58"] > .ww-image') ?

            setTimeout(() => {
                if (this.isPrerender && this.$el && this.$el.querySelector('.ww-image__img')) {
                    this.imgSrcSet = this.imgSrcSet || [];

                    const splited = this.wwElementState.uid.split('-');
                    const uid = splited[splited.length - 1];

                    const img = this.$el.querySelector('.ww-image__img');
                    let width = Math.round(img.getBoundingClientRect().width);

                    const transform = wwLib.getResponsiveStyleProp({
                        uid: this.wwElementState.uid,
                        states: [':anim'],
                        prop: 'transform',
                    });
                    if (transform && transform.scale) {
                        const scaleX = parseFloat(transform.scale.x);
                        if (scaleX !== 0) width = width / scaleX;
                        else {
                            width = 1200;
                        }
                    }

                    if (width) {
                        const airtablePrefix = 'https://dl.airtable.com/';
                        const privateFrenchFoundersPrefix = 'https://private.frenchfounders.com/';

                        let prefix = null;
                        let url = this.url;
                        if (!this.url.startsWith('http')) prefix = 'weweb';
                        else if (this.url.startsWith(airtablePrefix)) {
                            prefix = 'airtable/';
                            url = this.url.replace(airtablePrefix, '');
                        } else if (this.url.startsWith(privateFrenchFoundersPrefix)) {
                            prefix = 'private-frenchfounders/';
                            url = this.url.replace(privateFrenchFoundersPrefix, '');
                        }

                        //pixel-ratio: 1
                        const currentSrc = prefix
                            ? `${wwLib.wwUtils.transformToTwicPics(url, prefix)}/quality=90/resize=${Math.round(width)}`
                            : this.source;

                        const query = this.screenSizes[this.screenSize].query;

                        this.imgSrcSet.unshift({
                            src: currentSrc,
                            media: query ? `(${query})` : null,
                        });

                        //pixel-ratio: 2+
                        const currentSrcRatio = prefix
                            ? `${wwLib.wwUtils.transformToTwicPics(url, prefix)}/quality=90/resize=${Math.round(
                                  width * 2
                              )}`
                            : this.source;

                        this.imgSrcSet.unshift({
                            src: currentSrcRatio,
                            media: query
                                ? `(${query}) and (-webkit-min-device-pixel-ratio: 2)`
                                : '(-webkit-min-device-pixel-ratio: 2)',
                        });

                        let imgSrcSetElm = document.getElementById(`ww-image-srcset-${uid}`);
                        if (!imgSrcSetElm) {
                            imgSrcSetElm = document.createElement('script');
                            imgSrcSetElm.setAttribute('id', `ww-image-srcset-${uid}`);
                            document.head.append(imgSrcSetElm);
                        }
                        imgSrcSetElm.innerText = `window.wwg_imgsrcset_${uid} = ${JSON.stringify(this.imgSrcSet)};`;
                    }
                }
            }, 100);
        },
        /* wwManager:start */
        preventEvent(event) {
            event.preventDefault();
            event.stopPropagation();

            return false;
        },
        /*=============================================m_ÔÔ_m=============================================\
          IMAGE ZOOM
        \================================================================================================*/
        setRatio() {
            if (!this.$el) return;

            const img = this.$el.querySelector('img');
            if (!img) return;

            if (!img.naturalWidth || !img.naturalHeight) return;
            const ratio = img.naturalHeight / img.naturalWidth;
            this.$emit('update:content', { ratio });
        },
        resetZoom(event) {
            if (event) this.preventEvent(event);
            this.$emit('update:content', { x: 0, y: 0, zoom: 1 });

            return false;
        },
        switchCoverContain() {
            if (this.$el && this.$el.querySelector('img')) {
                const wrapper = this.$el.querySelector('.ww-image__wrapper');
                const wrapperWidth = wrapper.getBoundingClientRect().width;
                const wrapperHeight = wrapper.getBoundingClientRect().height;

                const img = this.$el.querySelector('img');
                const imgWidth = img.getBoundingClientRect().width;
                const imgHeight = img.getBoundingClientRect().height;

                let targetHeight, targetWidth;

                if (wrapperWidth === imgWidth) {
                    targetHeight = wrapperHeight;
                    targetWidth = imgWidth / (imgHeight / wrapperHeight);
                } else {
                    targetWidth = wrapperWidth;
                    targetHeight = imgHeight / (imgWidth / wrapperWidth);
                }

                const zoom = targetWidth / wrapperWidth;

                this.$emit('update:content', { x: 0, y: 0, zoom });
            }
        },

        /*=============================================m_ÔÔ_m=============================================\
          IMAGE MOVE
        \================================================================================================*/
        getEventPosition(event) {
            var position = { x: 0, y: 0 };

            position.x = event.clientX;
            position.y = event.clientY;

            return position;
        },
        startMove(event) {
            if (!this.isDoubleSelected) return;
            if (
                this.wwEditorState.editMode !== wwLib.wwEditorHelper.EDIT_MODES.EDITION ||
                this.wwElementState.isBackground
            ) {
                return;
            }
            if (event.ctrlKey || event.button == 2) {
                return;
            }
            this.isMoving = false;
            wwLib.wwManagerUI.lockSelection();

            this.lastMovePosition = this.getEventPosition(event);
            this.initialPosition = { x: this.content.x, y: this.content.y };
            if (this.lastMovePosition) {
                wwLib.getFrontDocument().addEventListener('mousemove', this.move);
                wwLib.getManagerDocument().addEventListener('mousemove', this.move);
                wwLib.getFrontDocument().addEventListener('click', this.stopMove);
                wwLib.getManagerDocument().addEventListener('click', this.stopMove);

                document.body.classList.add('ww-image-dragging');
                return false;
            }
        },
        move(event) {
            let position = this.getEventPosition(event);

            if (!position) {
                return;
            }

            var offsetXpx = position.x - this.lastMovePosition.x;
            var offsetYpx = position.y - this.lastMovePosition.y;

            if (!this.isMoving && Math.abs(offsetXpx) + Math.abs(offsetYpx) < 4) {
                return;
            }

            this.isMoving = true;

            if (this.moveDirection == 'x') {
                offsetYpx = 0;
            } else if (this.moveDirection == 'y') {
                offsetXpx = 0;
            } else if (wwLib.wwShortcuts.hasModifiers('SHIFT') && Math.abs(offsetXpx) > Math.abs(offsetYpx)) {
                offsetYpx = 0;
                this.moveDirection = 'x';
            } else if (wwLib.wwShortcuts.hasModifiers('SHIFT')) {
                offsetXpx = 0;
                this.moveDirection = 'y';
            }

            const rectImg = this.$el.querySelector('.ww-image__wrapper').getBoundingClientRect();
            var offsetXpercent = (offsetXpx * 100) / rectImg.width;
            var offsetYpercent = (offsetYpx * 100) / rectImg.height;

            const update = {
                x:
                    this.initialPosition.x +
                    (offsetXpercent * this.content.zoom) /
                        (1 - this.content.zoom + this.content.zoom * this.content.zoom),
                y:
                    this.initialPosition.y +
                    (offsetYpercent * this.content.zoom) /
                        (1 - this.content.zoom + this.content.zoom * this.content.zoom),
            };

            this.preventEvent(event);
            this.$emit('update:content', update);
        },
        stopMove() {
            this.isMoving = false;

            wwLib.getFrontDocument().removeEventListener('mousemove', this.move);
            wwLib.getManagerDocument().removeEventListener('mousemove', this.move);
            wwLib.getFrontDocument().removeEventListener('click', this.stopMove);
            wwLib.getManagerDocument().removeEventListener('click', this.stopMove);

            this.moveDirection = null;
            window.document.body.classList.remove('ww-image-dragging');

            wwLib.wwManagerUI.unlockSelection();

            return false;
        },
        /* wwManager:end */
    },
};
</script>

<style scoped lang="scss">
.ww-image {
    display: flex;
    position: relative;
    overflow: hidden;

    &__wrapper {
        width: 100%;
        position: relative;
        overflow: hidden;
        flex: 1;

        &:after {
            position: absolute;
            content: '';
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            background: var(--ww-image-overlay-background);
            pointer-events: none;
        }
    }

    &__ratio {
        visibility: hidden;
        position: relative;
        pointer-events: none;

        &:before {
            content: '';
            width: 1px;
            margin-left: -1px;
            float: left;
            height: 0;
            padding-top: var(--ww-image-ratio);
        }

        &:after {
            content: '';
            display: table;
            clear: both;
        }
    }

    &__img {
        position: absolute;
        --posX: calc(calc(1% * var(--left)) * calc(1 - calc(calc(1 - var(--zoom)) / 2)));
        --posY: calc(calc(1% * var(--top)) * calc(1 - calc(calc(1 - var(--zoom)) / 2)));
        top: calc(50% + var(--posY));
        left: calc(50% + var(--posX));
        width: calc(100% * var(--zoom));
        transform: translate(-50%, -50%);
        image-rendering: -webkit-optimize-contrast;

        &-picture {
            position: absolute;
            top: 0;
            left: 0;
            bottom: 0;
            right: 0;
        }
    }

    /* wwEditor:start */
    &__drag-overlay {
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background-color: var(--ww-color-dark-300);
        opacity: 0;
        z-index: 1;
        transition: opacity 0.3s ease;
        pointer-events: none;
        display: flex;
        justify-content: center;
        align-items: center;

        &-icon {
            width: 100px;
            height: 100px;
            max-width: 90%;
            max-height: 90%;
        }
    }
    &.-can-drag-image {
        .ww-image__drag-overlay {
            opacity: 0.3;
        }
        .ww-image__wrapper {
            cursor: move;
            cursor: grab;
            user-select: none;
        }
    }
    /* wwEditor:end */
}
</style>

<style lang="scss">
/* wwEditor:start */
.ww-image-dragging {
    cursor: move;
    cursor: grab;
    user-select: none;
}
/* wwEditor:end */
</style>
