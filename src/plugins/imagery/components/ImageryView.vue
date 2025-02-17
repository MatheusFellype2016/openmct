/*****************************************************************************
 * Open MCT, Copyright (c) 2014-2022, United States Government
 * as represented by the Administrator of the National Aeronautics and Space
 * Administration. All rights reserved.
 *
 * Open MCT is licensed under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 * http://www.apache.org/licenses/LICENSE-2.0.
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 * License for the specific language governing permissions and limitations
 * under the License.
 *
 * Open MCT includes source code licensed under additional open source
 * licenses. See the Open Source Licenses file (LICENSES.md) included with
 * this source code distribution or the Licensing information page available
 * at runtime from the About dialog for additional information.
 *****************************************************************************/

<template>
<div
    tabindex="0"
    class="c-imagery"
    @keyup="arrowUpHandler"
    @keydown="arrowDownHandler"
    @mouseover="focusElement"
>
    <div class="c-imagery__main-image-wrapper has-local-controls">
        <ImageControls
            ref="imageControls"
            :zoom-factor="zoomFactor"
            :image-url="imageUrl"
            @resetImage="resetImage"
            @panZoomUpdated="handlePanZoomUpdate"
            @filtersUpdated="setFilters"
            @cursorsUpdated="setCursorStates"
            @startPan="startPan"
        />

        <div
            ref="imageBG"
            class="c-imagery__main-image__bg"
            :class="{
                'paused unnsynced': isPaused && !isFixed,
                'stale': false,
                'pannable': cursorStates.isPannable,
                'cursor-zoom-in': cursorStates.showCursorZoomIn,
                'cursor-zoom-out': cursorStates.showCursorZoomOut
            }"
            @click="expand"
        >
            <div
                v-if="zoomFactor > 1"
                class="c-imagery__hints"
            >Alt-drag to pan</div>
            <div
                ref="focusedImageWrapper"
                class="image-wrapper"
                :style="{
                    'width': `${sizedImageWidth}px`,
                    'height': `${sizedImageHeight}px`
                }"
                @mousedown="handlePanZoomClick"
            >
                <img
                    ref="focusedImage"
                    class="c-imagery__main-image__image js-imageryView-image "
                    :src="imageUrl"
                    :draggable="!isSelectable"
                    :style="{
                        'filter': `brightness(${filters.brightness}%) contrast(${filters.contrast}%)`
                    }"
                    :data-openmct-image-timestamp="time"
                    :data-openmct-object-keystring="keyString"
                >
                <div
                    v-if="imageUrl"
                    ref="focusedImageElement"
                    class="c-imagery__main-image__background-image"
                    :draggable="!isSelectable"
                    :style="{
                        'filter': `brightness(${filters.brightness}%) contrast(${filters.contrast}%)`,
                        'background-image':
                            `${imageUrl ? (
                                `url(${imageUrl}),
                                    repeating-linear-gradient(
                                        45deg,
                                        transparent,
                                        transparent 4px,
                                        rgba(125,125,125,.2) 4px,
                                        rgba(125,125,125,.2) 8px
                                    )`
                            ) : ''}`,
                        'transform': `scale(${zoomFactor}) translate(${imageTranslateX}px, ${imageTranslateY}px)`,
                        'transition': `${!pan && animateZoom ? 'transform 250ms ease-in' : 'initial'}`,
                        'width': `${sizedImageWidth}px`,
                        'height': `${sizedImageHeight}px`,

                    }"
                ></div>
                <Compass
                    v-if="shouldDisplayCompass"
                    :compass-rose-sizing-classes="compassRoseSizingClasses"
                    :image="focusedImage"
                    :natural-aspect-ratio="focusedImageNaturalAspectRatio"
                    :sized-image-dimensions="sizedImageDimensions"
                />
            </div>
        </div>

        <button
            class="c-local-controls c-local-controls--show-on-hover c-imagery__prev-next-button c-nav c-nav--prev"
            title="Previous image"
            :disabled="isPrevDisabled"
            @click="prevImage()"
        ></button>

        <button
            class="c-local-controls c-local-controls--show-on-hover c-imagery__prev-next-button c-nav c-nav--next"
            title="Next image"
            :disabled="isNextDisabled"
            @click="nextImage()"
        ></button>

        <div class="c-imagery__control-bar">
            <div class="c-imagery__time">
                <div class="c-imagery__timestamp u-style-receiver js-style-receiver">{{ time }}</div>

                <!-- image fresh -->
                <div
                    v-if="canTrackDuration"
                    :class="{'c-imagery--new': isImageNew && !refreshCSS}"
                    class="c-imagery__age icon-timer"
                >{{ formattedDuration }}</div>

                <!-- spacecraft position fresh -->
                <div
                    v-if="relatedTelemetry.hasRelatedTelemetry && isSpacecraftPositionFresh"
                    class="c-imagery__age icon-check c-imagery--new"
                >POS</div>

                <!-- camera position fresh -->
                <div
                    v-if="relatedTelemetry.hasRelatedTelemetry && isCameraPositionFresh"
                    class="c-imagery__age icon-check c-imagery--new"
                >CAM</div>
            </div>
            <div class="h-local-controls">
                <button
                    v-if="!isFixed"
                    class="c-button icon-pause pause-play"
                    :class="{'is-paused': isPaused}"
                    @click="paused(!isPaused)"
                ></button>
            </div>
        </div>
    </div>
    <div
        class="c-imagery__thumbs-wrapper"
        :class="[
            { 'is-paused': isPaused && !isFixed },
            { 'is-autoscroll-off': !resizingWindow && !autoScroll && !isPaused }
        ]"
    >
        <div
            ref="thumbsWrapper"
            class="c-imagery__thumbs-scroll-area"
            @scroll="handleScroll"
        >
            <div
                v-for="(image, index) in imageHistory"
                :key="image.url + image.time"
                class="c-imagery__thumb c-thumb"
                :class="{ selected: focusedImageIndex === index && isPaused }"
                @click="thumbnailClicked(index)"
            >
                <a
                    href=""
                    :download="image.imageDownloadName"
                    @click.prevent
                >
                    <img
                        class="c-thumb__image"
                        :src="image.url"
                    >
                </a>
                <div class="c-thumb__timestamp">{{ image.formattedTime }}</div>
            </div>
        </div>

        <button
            class="c-imagery__auto-scroll-resume-button c-icon-button icon-play"
            title="Resume automatic scrolling of image thumbnails"
            @click="scrollToRight('reset')"
        ></button>
    </div>
</div>
</template>

<script>
import eventHelpers from '../lib/eventHelpers';
import _ from 'lodash';
import moment from 'moment';

import RelatedTelemetry from './RelatedTelemetry/RelatedTelemetry';
import Compass from './Compass/Compass.vue';
import ImageControls from './ImageControls.vue';
import imageryData from "../../imagery/mixins/imageryData";

const REFRESH_CSS_MS = 500;
const DURATION_TRACK_MS = 1000;
const ARROW_DOWN_DELAY_CHECK_MS = 400;
const ARROW_SCROLL_RATE_MS = 100;
const THUMBNAIL_CLICKED = true;

const ONE_MINUTE = 60 * 1000;
const FIVE_MINUTES = 5 * ONE_MINUTE;
const ONE_HOUR = ONE_MINUTE * 60;
const EIGHT_HOURS = 8 * ONE_HOUR;
const TWENTYFOUR_HOURS = EIGHT_HOURS * 3;

const ARROW_RIGHT = 39;
const ARROW_LEFT = 37;

const SCROLL_LATENCY = 250;

const ZOOM_SCALE_DEFAULT = 1;

export default {
    components: {
        Compass,
        ImageControls
    },
    mixins: [imageryData],
    inject: ['openmct', 'domainObject', 'objectPath', 'currentView'],
    props: {
        focusedImageTimestamp: {
            type: Number,
            default() {
                return undefined;
            }
        }
    },
    data() {
        let timeSystem = this.openmct.time.timeSystem();
        this.metadata = {};
        this.requestCount = 0;

        return {
            durationFormatter: undefined,
            imageHistory: [],
            timeSystem: timeSystem,
            keyString: undefined,
            autoScroll: true,
            thumbnailClick: THUMBNAIL_CLICKED,
            isPaused: false,
            refreshCSS: false,
            focusedImageIndex: undefined,
            focusedImageRelatedTelemetry: {},
            numericDuration: undefined,
            relatedTelemetry: {},
            latestRelatedTelemetry: {},
            focusedImageNaturalAspectRatio: undefined,
            imageContainerWidth: undefined,
            imageContainerHeight: undefined,
            sizedImageWidth: 0,
            sizedImageHeight: 0,
            lockCompass: true,
            resizingWindow: false,
            timeContext: undefined,
            zoomFactor: ZOOM_SCALE_DEFAULT,
            filters: {
                brightness: 100,
                contrast: 100
            },
            cursorStates: {
                isPannable: false,
                showCursorZoomIn: false,
                showCursorZoomOut: false,
                modifierKeyPressed: false
            },
            imageTranslateX: 0,
            imageTranslateY: 0,
            pan: undefined,
            animateZoom: true,
            imagePanned: false
        };
    },
    computed: {
        compassRoseSizingClasses() {
            let compassRoseSizingClasses = '';
            if (this.sizedImageWidth < 300) {
                compassRoseSizingClasses = '--rose-small --rose-min';
            } else if (this.sizedImageWidth < 500) {
                compassRoseSizingClasses = '--rose-small';
            } else if (this.sizedImageWidth > 1000) {
                compassRoseSizingClasses = '--rose-max';
            }

            return compassRoseSizingClasses;
        },
        time() {
            return this.formatTime(this.focusedImage);
        },
        imageUrl() {
            return this.formatImageUrl(this.focusedImage);
        },
        isImageNew() {
            let cutoff = FIVE_MINUTES;
            let age = this.numericDuration;

            return age < cutoff && !this.refreshCSS;
        },
        canTrackDuration() {
            let hasClock;
            if (this.timeContext) {
                hasClock = this.timeContext.clock();
            } else {
                hasClock = this.openmct.time.clock();
            }

            return hasClock && this.timeSystem.isUTCBased;
        },
        isNextDisabled() {
            let disabled = false;

            if (this.focusedImageIndex === -1 || this.focusedImageIndex === this.imageHistory.length - 1) {
                disabled = true;
            }

            return disabled;
        },
        isPrevDisabled() {
            let disabled = false;

            if (this.focusedImageIndex === 0 || this.imageHistory.length < 2) {
                disabled = true;
            }

            return disabled;
        },
        focusedImage() {
            return this.imageHistory[this.focusedImageIndex];
        },
        parsedSelectedTime() {
            return this.parseTime(this.focusedImage);
        },
        formattedDuration() {
            let result = 'N/A';
            let negativeAge = -1;

            if (this.numericDuration > TWENTYFOUR_HOURS) {
                negativeAge *= (this.numericDuration / TWENTYFOUR_HOURS);
                result = moment.duration(negativeAge, 'days').humanize(true);
            } else if (this.numericDuration > EIGHT_HOURS) {
                negativeAge *= (this.numericDuration / ONE_HOUR);
                result = moment.duration(negativeAge, 'hours').humanize(true);
            } else if (this.durationFormatter) {
                result = this.durationFormatter.format(this.numericDuration);
            }

            return result;
        },
        shouldDisplayCompass() {
            const imageHeightAndWidth = this.sizedImageHeight !== 0
                && this.sizedImageWidth !== 0;

            const display = this.focusedImage !== undefined
                && this.focusedImageNaturalAspectRatio !== undefined
                && this.imageContainerWidth !== undefined
                && this.imageContainerHeight !== undefined
                && imageHeightAndWidth
                && this.zoomFactor === 1
                && this.imagePanned !== true;

            return display;
        },
        isSpacecraftPositionFresh() {
            let isFresh = undefined;
            let latest = this.latestRelatedTelemetry;
            let focused = this.focusedImageRelatedTelemetry;

            if (this.relatedTelemetry.hasRelatedTelemetry) {
                isFresh = true;
                for (let key of this.spacecraftPositionKeys) {
                    if (this.relatedTelemetry[key] && latest[key] && focused[key]) {
                        isFresh = isFresh && Boolean(this.relatedTelemetry[key].comparisonFunction(latest[key], focused[key]));
                    } else {
                        isFresh = false;
                    }
                }
            }

            return isFresh;
        },
        isSpacecraftOrientationFresh() {
            let isFresh = undefined;
            let latest = this.latestRelatedTelemetry;
            let focused = this.focusedImageRelatedTelemetry;

            if (this.relatedTelemetry.hasRelatedTelemetry) {
                isFresh = true;
                for (let key of this.spacecraftOrientationKeys) {
                    if (this.relatedTelemetry[key] && latest[key] && focused[key]) {
                        isFresh = isFresh && Boolean(this.relatedTelemetry[key].comparisonFunction(latest[key], focused[key]));
                    } else {
                        isFresh = false;
                    }
                }
            }

            return isFresh;
        },
        isCameraPositionFresh() {
            let isFresh = undefined;
            let latest = this.latestRelatedTelemetry;
            let focused = this.focusedImageRelatedTelemetry;

            if (this.relatedTelemetry.hasRelatedTelemetry) {
                isFresh = true;

                // camera freshness relies on spacecraft position freshness
                if (this.isSpacecraftPositionFresh && this.isSpacecraftOrientationFresh) {
                    for (let key of this.cameraKeys) {
                        if (this.relatedTelemetry[key] && latest[key] && focused[key]) {
                            isFresh = isFresh && Boolean(this.relatedTelemetry[key].comparisonFunction(latest[key], focused[key]));
                        } else {
                            isFresh = false;
                        }
                    }
                } else {
                    isFresh = false;
                }
            }

            return isFresh;
        },
        isFixed() {
            let clock;
            if (this.timeContext) {
                clock = this.timeContext.clock();
            } else {
                clock = this.openmct.time.clock();
            }

            return clock === undefined;
        },
        isSelectable() {
            return true;

        },
        sizedImageDimensions() {
            return {
                width: this.sizedImageWidth,
                height: this.sizedImageHeight
            };
        }
    },
    watch: {
        imageHistory: {
            handler(newHistory, oldHistory) {
                const newSize = newHistory.length;
                let imageIndex;
                if (this.focusedImageTimestamp !== undefined) {
                    const foundImageIndex = newHistory.findIndex(img => img.time === this.focusedImageTimestamp);
                    imageIndex = foundImageIndex > -1
                        ? foundImageIndex
                        : newSize - 1;
                } else {
                    imageIndex = newSize > 0
                        ? newSize - 1
                        : undefined;
                }

                this.nextImageIndex = imageIndex;

                if (this.previousFocusedImage && newHistory.length) {
                    const matchIndex = this.matchIndexOfPreviousImage(
                        this.previousFocusedImage,
                        newHistory
                    );

                    if (matchIndex > -1) {
                        this.setFocusedImage(matchIndex);
                    } else {
                        this.paused();
                    }
                }

                if (!this.isPaused) {
                    this.setFocusedImage(imageIndex);
                    this.scrollToRight();
                }
            },
            deep: true
        },
        focusedImageIndex() {
            this.trackDuration();
            this.resetAgeCSS();
            this.updateRelatedTelemetryForFocusedImage();
            this.getImageNaturalDimensions();
        }
    },
    async mounted() {
        eventHelpers.extend(this);
        this.focusedImageWrapper = this.$refs.focusedImageWrapper;
        this.focusedImageElement = this.$refs.focusedImageElement;

        //We only need to use this till the user focuses an image manually
        if (this.focusedImageTimestamp !== undefined) {
            this.isPaused = true;
        }

        this.setTimeContext = this.setTimeContext.bind(this);
        this.setTimeContext();

        // related telemetry keys
        this.spacecraftPositionKeys = ['positionX', 'positionY', 'positionZ'];
        this.spacecraftOrientationKeys = ['heading'];
        this.cameraKeys = ['cameraPan', 'cameraTilt'];
        this.sunKeys = ['sunOrientation'];

        // related telemetry
        await this.initializeRelatedTelemetry();
        await this.updateRelatedTelemetryForFocusedImage();
        this.trackLatestRelatedTelemetry();

        // for scrolling through images quickly and resizing the object view
        this.updateRelatedTelemetryForFocusedImage = _.debounce(this.updateRelatedTelemetryForFocusedImage, 400);

        // for resizing the object view
        this.resizeImageContainer = _.debounce(this.resizeImageContainer, 400);

        if (this.$refs.imageBG) {
            this.imageContainerResizeObserver = new ResizeObserver(this.resizeImageContainer);
            this.imageContainerResizeObserver.observe(this.$refs.imageBG);
        }

        // For adjusting scroll bar size and position when resizing thumbs wrapper
        this.handleScroll = _.debounce(this.handleScroll, SCROLL_LATENCY);
        this.handleThumbWindowResizeEnded = _.debounce(this.handleThumbWindowResizeEnded, SCROLL_LATENCY);
        this.handleThumbWindowResizeStart = _.debounce(this.handleThumbWindowResizeStart, SCROLL_LATENCY);

        if (this.$refs.thumbsWrapper) {
            this.thumbWrapperResizeObserver = new ResizeObserver(this.handleThumbWindowResizeStart);
            this.thumbWrapperResizeObserver.observe(this.$refs.thumbsWrapper);
        }

        this.listenTo(this.focusedImageWrapper, 'wheel', this.wheelZoom, this);
    },
    beforeDestroy() {
        this.stopFollowingTimeContext();

        if (this.thumbWrapperResizeObserver) {
            this.thumbWrapperResizeObserver.disconnect();
        }

        if (this.imageContainerResizeObserver) {
            this.imageContainerResizeObserver.disconnect();
        }

        if (this.relatedTelemetry.hasRelatedTelemetry) {
            this.relatedTelemetry.destroy();
        }

        // unsubscribe from related telemetry
        if (this.relatedTelemetry.hasRelatedTelemetry) {
            for (let key of this.relatedTelemetry.keys) {
                if (this.relatedTelemetry[key] && this.relatedTelemetry[key].unsubscribe) {
                    this.relatedTelemetry[key].unsubscribe();
                }
            }
        }

        this.stopListening(this.focusedImageWrapper, 'wheel', this.wheelZoom, this);

    },
    methods: {
        setTimeContext() {
            this.stopFollowingTimeContext();
            this.timeContext = this.openmct.time.getContextForView(this.objectPath);
            //listen
            this.timeContext.on('timeSystem', this.trackDuration);
            this.timeContext.on('clock', this.trackDuration);
        },
        stopFollowingTimeContext() {
            if (this.timeContext) {
                this.timeContext.off("timeSystem", this.trackDuration);
                this.timeContext.off("clock", this.trackDuration);
            }
        },
        expand() {
            // check for modifier keys so it doesnt interfere with the layout
            if (this.cursorStates.modifierKeyPressed) {
                return;
            }

            const actionCollection = this.openmct.actions.getActionsCollection(this.objectPath, this.currentView);
            const visibleActions = actionCollection.getVisibleActions();
            const viewLargeAction = visibleActions
                && visibleActions.find(action => action.key === 'large.view');

            if (viewLargeAction && viewLargeAction.appliesTo(this.objectPath, this.currentView)) {
                viewLargeAction.onItemClicked();
            }
        },
        async initializeRelatedTelemetry() {
            this.relatedTelemetry = new RelatedTelemetry(
                this.openmct,
                this.domainObject,
                [...this.spacecraftPositionKeys, ...this.spacecraftOrientationKeys, ...this.cameraKeys, ...this.sunKeys]
            );

            if (this.relatedTelemetry.hasRelatedTelemetry) {
                await this.relatedTelemetry.load();
            }
        },
        async getMostRecentRelatedTelemetry(key, targetDatum) {
            if (!this.relatedTelemetry.hasRelatedTelemetry) {
                throw new Error(`${this.domainObject.name} does not have any related telemetry`);
            }

            if (!this.relatedTelemetry[key]) {
                throw new Error(`${key} does not exist on related telemetry`);
            }

            let mostRecent;
            let valueKey = this.relatedTelemetry[key].historical.valueKey;
            let valuesOnTelemetry = this.relatedTelemetry[key].hasTelemetryOnDatum;

            if (valuesOnTelemetry) {
                mostRecent = targetDatum[valueKey];

                if (mostRecent) {
                    return mostRecent;
                } else {
                    console.warn(`Related Telemetry for ${key} does NOT exist on this telemetry datum as configuration implied.`);

                    return;
                }
            }

            mostRecent = await this.relatedTelemetry[key].requestLatestFor(targetDatum);

            return mostRecent[valueKey];
        },
        // will subscribe to data for this key if not already done
        subscribeToDataForKey(key) {
            if (this.relatedTelemetry[key].isSubscribed) {
                return;
            }

            if (this.relatedTelemetry[key].realtimeDomainObject) {
                this.relatedTelemetry[key].unsubscribe = this.openmct.telemetry.subscribe(
                    this.relatedTelemetry[key].realtimeDomainObject, datum => {
                        this.relatedTelemetry[key].listeners.forEach(callback => {
                            callback(datum);
                        });

                    }
                );

                this.relatedTelemetry[key].isSubscribed = true;
            }
        },
        async updateRelatedTelemetryForFocusedImage() {
            if (!this.relatedTelemetry.hasRelatedTelemetry || !this.focusedImage) {
                return;
            }

            // set data ON image telemetry as well as in focusedImageRelatedTelemetry
            for (let key of this.relatedTelemetry.keys) {
                if (
                    this.relatedTelemetry[key]
                    && this.relatedTelemetry[key].historical
                    && this.relatedTelemetry[key].requestLatestFor

                ) {
                    let valuesOnTelemetry = this.relatedTelemetry[key].hasTelemetryOnDatum;
                    let value = await this.getMostRecentRelatedTelemetry(key, this.focusedImage);

                    if (!valuesOnTelemetry) {
                        this.$set(this.imageHistory[this.focusedImageIndex], key, value); // manually add to telemetry
                    }

                    this.$set(this.focusedImageRelatedTelemetry, key, value);
                }
            }
        },
        trackLatestRelatedTelemetry() {
            [...this.spacecraftPositionKeys, ...this.spacecraftOrientationKeys, ...this.cameraKeys, ...this.sunKeys].forEach(key => {
                if (this.relatedTelemetry[key] && this.relatedTelemetry[key].subscribe) {
                    this.relatedTelemetry[key].subscribe((datum) => {
                        let valueKey = this.relatedTelemetry[key].realtime.valueKey;
                        this.$set(this.latestRelatedTelemetry, key, datum[valueKey]);
                    });
                }
            });
        },
        focusElement() {
            this.$el.focus();
        },

        handleScroll() {
            const thumbsWrapper = this.$refs.thumbsWrapper;
            if (!thumbsWrapper || this.resizingWindow) {
                return;
            }

            const { scrollLeft, scrollWidth, clientWidth } = thumbsWrapper;
            const disableScroll = scrollWidth > Math.ceil(scrollLeft + clientWidth);
            this.autoScroll = !disableScroll;
        },
        paused(state) {
            this.isPaused = Boolean(state);

            if (!state) {
                this.previousFocusedImage = null;
                this.setFocusedImage(this.nextImageIndex);
                this.autoScroll = true;
                this.scrollToRight();
            }
        },
        scrollToFocused() {
            const thumbsWrapper = this.$refs.thumbsWrapper;
            if (!thumbsWrapper) {
                return;
            }

            let domThumb = thumbsWrapper.children[this.focusedImageIndex];

            if (domThumb) {
                domThumb.scrollIntoView({
                    behavior: 'smooth',
                    block: 'center'
                });
            }
        },
        scrollToRight(type) {
            if (type !== 'reset' && (this.isPaused || !this.$refs.thumbsWrapper || !this.autoScroll)) {
                return;
            }

            const scrollWidth = this.$refs.thumbsWrapper.scrollWidth || 0;
            if (!scrollWidth) {
                return;
            }

            this.$nextTick(() => {
                this.$refs.thumbsWrapper.scrollLeft = scrollWidth;
            });
        },
        matchIndexOfPreviousImage(previous, imageHistory) {
            // match logic uses a composite of url and time to account
            // for example imagery not having fully unique urls
            return imageHistory.findIndex((x) => (
                x.url === previous.url
                && x.time === previous.time
            ));
        },
        thumbnailClicked(index) {
            this.setFocusedImage(index);
            this.paused(true);

            this.setPreviousFocusedImage(index);
        },
        setPreviousFocusedImage(index) {
            this.focusedImageTimestamp = undefined;
            this.previousFocusedImage = this.imageHistory[index]
                ? JSON.parse(JSON.stringify(this.imageHistory[index]))
                : undefined;
        },
        setFocusedImage(index) {
            if (!(Number.isInteger(index) && index > -1)) {
                return;
            }

            this.focusedImageIndex = index;
        },
        trackDuration() {
            if (this.canTrackDuration) {
                this.stopDurationTracking();
                this.updateDuration();
                this.durationTracker = window.setInterval(
                    this.updateDuration, DURATION_TRACK_MS
                );
            } else {
                this.stopDurationTracking();
            }
        },
        stopDurationTracking() {
            window.clearInterval(this.durationTracker);
        },
        updateDuration() {
            let currentTime = this.timeContext.clock() && this.timeContext.clock().currentValue();
            if (currentTime === undefined) {
                this.numericDuration = currentTime;
            } else {
                this.numericDuration = currentTime - this.parsedSelectedTime;
            }
        },
        resetAgeCSS() {
            this.refreshCSS = true;
            // unable to make this work with nextTick
            setTimeout(() => {
                this.refreshCSS = false;
            }, REFRESH_CSS_MS);
        },
        nextImage() {
            if (this.isNextDisabled) {
                return;
            }

            let index = this.focusedImageIndex;

            this.thumbnailClicked(++index);
            if (index === this.imageHistory.length - 1) {
                this.paused(false);
            }
        },
        prevImage() {
            if (this.isPrevDisabled) {
                return;
            }

            let index = this.focusedImageIndex;

            if (index === this.imageHistory.length - 1) {
                this.thumbnailClicked(this.imageHistory.length - 2);
            } else {
                this.thumbnailClicked(--index);
            }
        },
        resetImage() {
            this.imagePanned = false;
            this.zoomFactor = ZOOM_SCALE_DEFAULT;
            this.imageTranslateX = 0;
            this.imageTranslateY = 0;
        },
        handlePanZoomUpdate({ newScaleFactor, screenClientX, screenClientY }) {
            if (!this.isPaused) {
                this.paused(true);
            }

            if (!(screenClientX || screenClientY)) {
                return this.updatePanZoom(newScaleFactor, 0, 0);
            }

            // handle mouse events
            const imageRect = this.focusedImageWrapper.getBoundingClientRect();
            const imageContainerX = screenClientX - imageRect.left;
            const imageContainerY = screenClientY - imageRect.top;
            const offsetFromCenterX = (imageRect.width / 2) - imageContainerX;
            const offsetFromCenterY = (imageRect.height / 2) - imageContainerY;

            this.updatePanZoom(newScaleFactor, offsetFromCenterX, offsetFromCenterY);
        },
        updatePanZoom(newScaleFactor, offsetFromCenterX, offsetFromCenterY) {
            const currentScale = this.zoomFactor;
            const previousTranslateX = this.imageTranslateX;
            const previousTranslateY = this.imageTranslateY;

            const offsetXInOriginalScale = offsetFromCenterX / currentScale;
            const offsetYInOriginalScale = offsetFromCenterY / currentScale;
            const translateX = offsetXInOriginalScale + previousTranslateX;
            const translateY = offsetYInOriginalScale + previousTranslateY;
            this.imageTranslateX = translateX;
            this.imageTranslateY = translateY;
            this.zoomFactor = newScaleFactor;
        },
        handlePanZoomClick(e) {
            this.$refs.imageControls.handlePanZoomClick(e);
        },
        arrowDownHandler(event) {
            let key = event.keyCode;

            if (this.isLeftOrRightArrowKey(key)) {
                this.arrowDown = true;
                window.clearTimeout(this.arrowDownDelayTimeout);
                this.arrowDownDelayTimeout = window.setTimeout(() => {
                    this.arrowKeyScroll(this.directionByKey(key));
                }, ARROW_DOWN_DELAY_CHECK_MS);
            }
        },
        arrowUpHandler(event) {
            let key = event.keyCode;

            window.clearTimeout(this.arrowDownDelayTimeout);

            if (this.isLeftOrRightArrowKey(key)) {
                this.arrowDown = false;
                let direction = this.directionByKey(key);
                this[direction + 'Image']();
            }
        },
        arrowKeyScroll(direction) {
            if (this.arrowDown) {
                this.arrowKeyScrolling = true;
                this[direction + 'Image']();
                setTimeout(() => {
                    this.arrowKeyScroll(direction);
                }, ARROW_SCROLL_RATE_MS);
            } else {
                window.clearTimeout(this.arrowDownDelayTimeout);
                this.arrowKeyScrolling = false;
                this.scrollToFocused();
            }
        },
        directionByKey(keyCode) {
            let direction;

            if (keyCode === ARROW_LEFT) {
                direction = 'prev';
            }

            if (keyCode === ARROW_RIGHT) {
                direction = 'next';
            }

            return direction;
        },
        isLeftOrRightArrowKey(keyCode) {
            return [ARROW_RIGHT, ARROW_LEFT].includes(keyCode);
        },
        getImageNaturalDimensions() {
            this.focusedImageNaturalAspectRatio = undefined;

            const img = this.$refs.focusedImage;
            if (!img) {
                return;
            }

            // TODO - should probably cache this
            img.addEventListener('load', () => {
                this.setSizedImageDimensions();
            }, { once: true });
        },
        resizeImageContainer() {
            if (!this.$refs.imageBG) {
                return;
            }

            if (this.$refs.imageBG.clientWidth !== this.imageContainerWidth) {
                this.imageContainerWidth = this.$refs.imageBG.clientWidth;
            }

            if (this.$refs.imageBG.clientHeight !== this.imageContainerHeight) {
                this.imageContainerHeight = this.$refs.imageBG.clientHeight;
            }

            this.setSizedImageDimensions();
        },
        setSizedImageDimensions() {
            this.focusedImageNaturalAspectRatio = this.$refs.focusedImage.naturalWidth / this.$refs.focusedImage.naturalHeight;

            if ((this.imageContainerWidth / this.imageContainerHeight) > this.focusedImageNaturalAspectRatio) {
                // container is wider than image
                this.sizedImageWidth = this.imageContainerHeight * this.focusedImageNaturalAspectRatio;
                this.sizedImageHeight = this.imageContainerHeight;
            } else {
                // container is taller than image
                this.sizedImageWidth = this.imageContainerWidth;
                this.sizedImageHeight = this.imageContainerWidth / this.focusedImageNaturalAspectRatio;
            }
        },
        handleThumbWindowResizeStart() {
            if (!this.autoScroll) {
                return;
            }

            // To hide resume button while scrolling
            this.resizingWindow = true;
            this.handleThumbWindowResizeEnded();
        },
        handleThumbWindowResizeEnded() {
            if (!this.isPaused) {
                this.scrollToRight('reset');
            }

            this.$nextTick(() => {
                this.resizingWindow = false;
            });
        },
        // debounced method
        clearWheelZoom() {
            this.$refs.imageControls.clearWheelZoom();
        },
        wheelZoom(e) {
            e.preventDefault();
            if (!this.isPaused) {
                this.paused(true);
            }

            this.$refs.imageControls.wheelZoom(e);
        },
        startPan(e) {
            e.preventDefault();
            if (!this.pan && this.zoomFactor > 1) {
                this.animateZoom = false;
                this.imagePanned = true;
                this.pan = {
                    x: e.clientX,
                    y: e.clientY
                };
                this.listenTo(window, 'mouseup', this.onMouseUp, this);
                this.listenTo(window, 'mousemove', this.trackMousePosition, this);
            }

            return false;
        },
        trackMousePosition(e) {
            if (!e.altKey) {
                return this.onMouseUp(e);
            }

            this.updatePan(e);
            e.preventDefault();

        },
        updatePan(e) {
            if (!this.pan) {
                return;
            }

            const dX = e.clientX - this.pan.x;
            const dY = e.clientY - this.pan.y;
            this.pan = {
                x: e.clientX,
                y: e.clientY
            };
            this.updatePanZoom(this.zoomFactor, dX, dY);
        },
        endPan() {
            this.pan = undefined;
            this.animateZoom = true;
        },
        onMouseUp(event) {
            this.stopListening(window, 'mouseup', this.onMouseUp, this);
            this.stopListening(window, 'mousemove', this.trackMousePosition, this);

            if (this.pan) {
                return this.endPan(event);
            }
        },
        setFilters(filtersObj) {
            this.filters = filtersObj;
        },
        setCursorStates(states) {
            this.cursorStates = states;
        }
    }
};
</script>
