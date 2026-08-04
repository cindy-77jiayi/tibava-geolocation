<template>
  <section v-if="variant === 'map'" class="geo-annotation pa-4">
    <div class="d-flex align-start justify-space-between mb-3">
      <div>
        <div class="text-subtitle-1 font-weight-bold">Map</div>
        <div class="text-caption grey--text text--darken-1">
          Mock data for UI demonstration — not model output
        </div>
      </div>
      <v-chip small outlined color="deep-purple">{{ timecode(currentTime) }}</v-chip>
    </div>

    <div
      ref="mapFrame"
      class="map-frame"
      :class="{ 'map-frame--draggable': mapZoom > 1, 'map-frame--dragging': mapDrag }"
      :style="{ '--active-color': colorFor(currentSegment.tag) }"
      @wheel.prevent="zoomFromWheel"
      @pointerdown="startMapPan"
      @pointermove="moveMapPan"
      @pointerup="endMapPan"
      @pointercancel="endMapPan"
      @pointerleave="endMapPan"
    >
      <svg class="map" :style="{ transform: `translate(${mapPan.x}px, ${mapPan.y}px) scale(${mapZoom})` }" viewBox="0 0 720 360" role="img" :aria-label="mapAriaLabel">
        <defs>
          <pattern id="map-grid" width="60" height="45" patternUnits="userSpaceOnUse">
            <path d="M 60 0 L 0 0 0 45" fill="none" stroke="#b9c5d6" stroke-width="0.8" opacity="0.7" />
          </pattern>
          <filter id="marker-shadow" x="-80%" y="-80%" width="260%" height="260%">
            <feDropShadow dx="0" dy="2" stdDeviation="2" flood-opacity="0.35" />
          </filter>
        </defs>
        <rect width="720" height="360" fill="#dbeafe" />
        <rect width="720" height="360" fill="url(#map-grid)" />
        <path class="continent" d="M105 77 L142 49 185 58 204 85 185 112 194 139 171 158 152 145 143 117 120 104Z" />
        <path class="continent" d="M195 178 L224 166 251 188 243 218 262 247 244 299 218 310 209 265 190 235Z" />
        <path class="continent" d="M338 76 L382 54 438 61 471 86 456 111 418 115 399 101 377 118 349 106Z" />
        <path class="continent" d="M410 126 L449 134 478 165 470 203 492 227 467 273 435 293 417 257 425 222 400 190 384 159Z" />
        <path class="continent" d="M510 211 L548 198 579 219 574 257 550 278 518 263 501 238Z" />
        <path class="continent" d="M599 102 L626 112 635 137 620 148 600 135Z" />

        <g
          v-for="location in mapLocations"
          :key="location.tag"
          class="map-location"
          @mouseenter="showLocation(location)"
          @mouseleave="hideLocation"
        >
          <circle :cx="markerX(location)" :cy="markerY(location)" r="4" :fill="location.color" opacity="0.45" />
        </g>

        <g
          v-if="currentSegment.tag"
          class="map-current-marker"
          filter="url(#marker-shadow)"
          @mouseenter="showLocation(currentSegment)"
          @mouseleave="hideLocation"
        >
          <ellipse
            :cx="markerX(currentSegment)"
            :cy="markerY(currentSegment)"
            rx="32"
            ry="22"
            :fill="colorFor(currentSegment.tag)"
            :opacity="heatOpacity(currentSegment)"
          />
          <circle :cx="markerX(currentSegment)" :cy="markerY(currentSegment)" r="10" fill="white" :stroke="colorFor(currentSegment.tag)" stroke-width="5" />
          <circle :cx="markerX(currentSegment)" :cy="markerY(currentSegment)" r="3" :fill="colorFor(currentSegment.tag)" />
        </g>
      </svg>
      <div
        v-if="hoveredLocation"
        class="map-tooltip"
        :class="{ 'map-tooltip--right': mapTooltipX(hoveredLocation) > 80 }"
        :style="mapTooltipStyle(hoveredLocation)"
      >
        <strong>{{ hoveredLocation.location }}</strong>
        <span>{{ formatCoordinates(hoveredLocation) }}</span>
      </div>
      <div class="map-controls" aria-label="Map zoom controls">
        <v-btn icon small aria-label="Zoom in" title="Zoom in" @click="zoomIn">
          <v-icon small>mdi-plus</v-icon>
        </v-btn>
        <v-btn icon small aria-label="Zoom out" title="Zoom out" :disabled="mapZoom === 1" @click="zoomOut">
          <v-icon small>mdi-minus</v-icon>
        </v-btn>
        <v-btn icon small aria-label="Reset to world view" title="Reset to world view" :disabled="mapZoom === 1" @click="resetMap">
          <v-icon small>mdi-earth</v-icon>
        </v-btn>
      </div>
      <div class="map-confidence">{{ currentSegment.tag ? `${currentSegment.tag} · ${confidenceLabel(currentSegment.confidence)} confidence` : "No prediction" }}</div>
    </div>
  </section>

  <section v-else class="geo-timeline px-4 pb-4">
    <div class="d-flex align-center justify-space-between mb-2">
      <div>
        <div class="text-subtitle-1 font-weight-bold">Geolocation timeline</div>
        <div class="text-caption grey--text text--darken-1">One mock timeline · each block is a shot · colour = place/tag · opacity = confidence</div>
      </div>
      <div class="geo-legend">
        <span v-for="scene in sceneLegend" :key="scene.tag"><i :style="{ backgroundColor: scene.color }"></i>{{ scene.tag }}</span>
      </div>
    </div>

    <div ref="timeline" class="geo-timeline__track" role="slider" aria-label="Mock geolocation timeline" @click="seekFromTimeline">
      <button
        v-for="segment in segments"
        :key="segment.id"
        class="geo-timeline__segment"
        :class="{ 'geo-timeline__segment--current': currentSegment.id === segment.id, 'geo-timeline__segment--unlabelled': !segment.tag }"
        :style="segmentStyle(segment)"
        type="button"
        :aria-label="segmentAriaLabel(segment)"
        @click.stop="jumpToSegment(segment)"
      >
      </button>
      <div class="geo-timeline__playhead" :style="{ left: `${playheadPercent}%` }">
        <span></span>
      </div>
    </div>
    <div class="geo-timeline__ticks">
      <span v-for="tick in timelineTicks" :key="tick">{{ timecode(tick) }}</span>
    </div>
    <div class="text-caption mt-2">
      <template v-if="currentSegment.tag">Selected: <strong>{{ currentSegment.tag }}</strong> · {{ currentSegment.location }} · {{ confidenceLabel(currentSegment.confidence) }} confidence</template>
      <template v-else>Selected: <strong>Unlabelled shot</strong> · no location prediction</template>
    </div>
  </section>
</template>

<script>
import { mapStores } from "pinia";
import { usePlayerStore } from "@/store/player";

const LOCATION_BY_TAG = {
  Indoor: { latitude: 48.8566, longitude: 2.3522, location: "Paris", color: "#4f46e5" },
  Outdoor: { latitude: -33.8688, longitude: 151.2093, location: "Sydney", color: "#ea580c" },
};

const SHOT_TEMPLATE = [
  { start: 0, end: 0.07, tag: "Outdoor", confidence: 0.98 },
  { start: 0.07, end: 0.14, tag: null },
  { start: 0.14, end: 0.25, tag: "Indoor", confidence: 0.28 },
  { start: 0.25, end: 0.31, tag: null },
  { start: 0.31, end: 0.38, tag: "Indoor", confidence: 0.84 },
  { start: 0.38, end: 0.48, tag: "Outdoor", confidence: 0.52 },
  { start: 0.48, end: 0.58, tag: null },
  { start: 0.58, end: 0.62, tag: "Outdoor", confidence: 0.16 },
  { start: 0.62, end: 0.75, tag: "Indoor", confidence: 0.73 },
  { start: 0.75, end: 0.84, tag: null },
  { start: 0.84, end: 0.93, tag: "Outdoor", confidence: 0.95 },
  { start: 0.93, end: 1, tag: null },
];

export default {
  data() {
    return {
      hoveredLocation: null,
      mapZoom: 1,
      mapPan: { x: 0, y: 0 },
      mapDrag: null,
    };
  },
  props: {
    variant: {
      type: String,
      default: "map",
      validator: (value) => ["map", "timeline"].includes(value),
    },
  },
  computed: {
    duration() {
      return this.playerStore.videoDuration > 0 ? this.playerStore.videoDuration : 15.482;
    },
    currentTime() {
      return Math.min(Math.max(this.playerStore.currentTime || 0, 0), this.duration);
    },
    segments() {
      return SHOT_TEMPLATE.map((item, index) => ({
        ...item,
        ...(item.tag ? LOCATION_BY_TAG[item.tag] : {}),
        id: `mock-geo-${index}`,
        start: item.start * this.duration,
        end: item.end * this.duration,
      }));
    },
    currentSegment() {
      return this.segments.find((segment) => this.currentTime >= segment.start && this.currentTime < segment.end) || this.segments[this.segments.length - 1];
    },
    sceneLegend() {
      return [
        { tag: "Indoor", color: LOCATION_BY_TAG.Indoor.color, icon: "mdi-home-city-outline" },
        { tag: "Outdoor", color: LOCATION_BY_TAG.Outdoor.color, icon: "mdi-tree-outline" },
      ];
    },
    mapLocations() {
      return Object.entries(LOCATION_BY_TAG).map(([tag, location]) => ({ tag, ...location }));
    },
    mapAriaLabel() {
      return this.currentSegment.tag ? `Mock map annotation at ${this.currentSegment.location}` : "Mock map annotation with no location prediction for the selected shot";
    },
    playheadPercent() {
      return (this.currentTime / this.duration) * 100;
    },
    timelineTicks() {
      return [0, this.duration * 0.25, this.duration * 0.5, this.duration * 0.75, this.duration];
    },
    ...mapStores(usePlayerStore),
  },
  methods: {
    colorFor(tag) {
      return tag ? LOCATION_BY_TAG[tag].color : "transparent";
    },
    markerX(segment) {
      return ((segment.longitude + 180) / 360) * 720;
    },
    markerY(segment) {
      return ((90 - segment.latitude) / 180) * 360;
    },
    mapTooltipX(location) {
      const frameWidth = this.$refs.mapFrame ? this.$refs.mapFrame.clientWidth : 720;
      return 50 + ((this.markerX(location) / 720) * 100 - 50) * this.mapZoom + (this.mapPan.x / frameWidth) * 100;
    },
    mapTooltipStyle(location) {
      const x = this.mapTooltipX(location);
      const frameHeight = this.$refs.mapFrame ? this.$refs.mapFrame.clientHeight : 360;
      const y = 50 + ((this.markerY(location) / 360) * 100 - 50) * this.mapZoom + (this.mapPan.y / frameHeight) * 100;
      return { left: `${x}%`, top: `${y}%` };
    },
    heatOpacity(segment) {
      return 0.06 + segment.confidence * 0.8;
    },
    segmentStyle(segment) {
      const isLabelled = Boolean(segment.tag);
      return {
        left: `${(segment.start / this.duration) * 100}%`,
        width: `${((segment.end - segment.start) / this.duration) * 100}%`,
        backgroundColor: isLabelled ? this.colorFor(segment.tag) : "transparent",
        opacity: isLabelled ? 0.08 + segment.confidence * 0.92 : 1,
      };
    },
    segmentWidth(segment) {
      return ((segment.end - segment.start) / this.duration) * 100;
    },
    confidenceLabel(confidence) {
      return `${Math.round(confidence * 100)}%`;
    },
    timecode(seconds) {
      const minutes = Math.floor(seconds / 60);
      const remainingSeconds = Math.floor(seconds % 60);
      return `${String(minutes).padStart(2, "0")}:${String(remainingSeconds).padStart(2, "0")}`;
    },
    formatCoordinates(segment) {
      return `${segment.latitude.toFixed(2)}°, ${segment.longitude.toFixed(2)}°`;
    },
    segmentAriaLabel(segment) {
      const interval = `${this.timecode(segment.start)} to ${this.timecode(segment.end)}`;
      return segment.tag ? `${segment.tag}, ${this.confidenceLabel(segment.confidence)} confidence, ${interval}` : `Unlabelled shot, ${interval}`;
    },
    setTime(time) {
      const safeTime = Math.min(Math.max(time, 0), this.duration);
      this.playerStore.setCurrentTime(safeTime);
      this.playerStore.setTargetTime(safeTime);
    },
    jumpToSegment(segment) {
      this.setTime((segment.start + segment.end) / 2);
    },
    showLocation(location) {
      this.hoveredLocation = location;
    },
    hideLocation() {
      this.hoveredLocation = null;
    },
    setMapZoom(zoom) {
      this.mapZoom = Math.min(Math.max(Number(zoom.toFixed(1)), 1), 3);
      this.setMapPan(this.mapPan.x, this.mapPan.y);
    },
    zoomIn() {
      this.setMapZoom(this.mapZoom + 0.25);
    },
    zoomOut() {
      this.setMapZoom(this.mapZoom - 0.25);
    },
    resetMap() {
      this.mapZoom = 1;
      this.mapPan = { x: 0, y: 0 };
    },
    zoomFromWheel(event) {
      this.setMapZoom(this.mapZoom + (event.deltaY < 0 ? 0.25 : -0.25));
    },
    mapPanLimits() {
      const frame = this.$refs.mapFrame;
      if (!frame) return { x: 0, y: 0 };
      return {
        x: (frame.clientWidth * (this.mapZoom - 1)) / 2,
        y: (frame.clientHeight * (this.mapZoom - 1)) / 2,
      };
    },
    setMapPan(x, y) {
      const limits = this.mapPanLimits();
      this.mapPan = {
        x: Math.min(Math.max(x, -limits.x), limits.x),
        y: Math.min(Math.max(y, -limits.y), limits.y),
      };
    },
    startMapPan(event) {
      if (this.mapZoom <= 1 || event.button !== 0 || event.target.closest(".map-controls")) return;
      event.preventDefault();
      event.currentTarget.setPointerCapture(event.pointerId);
      this.mapDrag = {
        startX: event.clientX,
        startY: event.clientY,
        panX: this.mapPan.x,
        panY: this.mapPan.y,
      };
    },
    moveMapPan(event) {
      if (!this.mapDrag) return;
      this.setMapPan(this.mapDrag.panX + event.clientX - this.mapDrag.startX, this.mapDrag.panY + event.clientY - this.mapDrag.startY);
    },
    endMapPan(event) {
      if (!this.mapDrag) return;
      if (event.currentTarget.hasPointerCapture(event.pointerId)) event.currentTarget.releasePointerCapture(event.pointerId);
      this.mapDrag = null;
    },
    seekFromTimeline(event) {
      const bounds = this.$refs.timeline.getBoundingClientRect();
      this.setTime(((event.clientX - bounds.left) / bounds.width) * this.duration);
    },
  },
};
</script>

<style scoped>
.geo-annotation { height: 100%; overflow: hidden; }
.map-frame { position: relative; overflow: hidden; border: 1px solid #d7dfec; border-radius: 10px; background: #dbeafe; box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.5); }
.map-frame--draggable { cursor: grab; touch-action: none; }
.map-frame--dragging { cursor: grabbing; }
.map { display: block; width: 100%; height: auto; aspect-ratio: 2 / 1; transform-origin: center; transition: transform .18s ease-out; }
.continent { fill: #f8fafc; stroke: #b8c4d6; stroke-width: 1.4; }
.map-location, .map-current-marker { cursor: help; }
.map-frame--dragging .map-location, .map-frame--dragging .map-current-marker { cursor: grabbing; }
.map-tooltip { position: absolute; z-index: 2; min-width: 122px; padding: 7px 9px; border-radius: 6px; background: rgba(15, 23, 42, 0.9); color: white; font-size: 12px; line-height: 1.35; pointer-events: none; transform: translate(-50%, calc(-100% - 10px)); box-shadow: 0 2px 8px rgba(15, 23, 42, 0.25); }
.map-tooltip--right { transform: translate(calc(-100% + 10px), calc(-100% - 10px)); }
.map-tooltip strong, .map-tooltip span { display: block; }
.map-controls { position: absolute; z-index: 3; bottom: 10px; left: 10px; display: flex; gap: 2px; padding: 2px; border: 1px solid rgba(148, 163, 184, .72); border-radius: 7px; background: rgba(255, 255, 255, .9); box-shadow: 0 1px 5px rgba(15, 23, 42, .18); }
.map-controls .v-btn { min-width: 28px; width: 28px; height: 28px; }
.map-confidence { position: absolute; padding: 5px 8px; border-radius: 5px; background: rgba(15, 23, 42, 0.78); color: white; font-size: 12px; backdrop-filter: blur(3px); }
.map-confidence { right: 10px; top: 10px; }
.geo-timeline { width: 100%; }
.geo-legend { display: flex; flex-wrap: wrap; gap: 12px; font-size: 12px; }
.geo-legend span { display: inline-flex; align-items: center; gap: 4px; }
.geo-legend i { display: inline-block; width: 9px; height: 9px; border-radius: 2px; }
.geo-timeline__track { position: relative; height: 52px; overflow: hidden; border: 1px solid #cbd5e1; border-radius: 6px; background: #e2e8f0; cursor: crosshair; }
.geo-timeline__segment { position: absolute; top: 0; bottom: 0; display: flex; align-items: center; justify-content: center; overflow: hidden; border: 0; border-right: 2px solid rgba(255, 255, 255, 0.85); color: white; font-size: 12px; font-weight: 700; text-shadow: 0 1px 2px rgba(15, 23, 42, 0.35); cursor: pointer; transition: filter .16s ease, box-shadow .16s ease; }
.geo-timeline__segment:hover { filter: brightness(1.06); }
.geo-timeline__segment--current { z-index: 1; box-shadow: inset 0 0 0 3px #0f172a; }
.geo-timeline__playhead { position: absolute; z-index: 3; top: 0; bottom: 0; width: 2px; background: #0f172a; pointer-events: none; }
.geo-timeline__playhead span { position: absolute; top: 0; left: -5px; width: 12px; height: 12px; background: #0f172a; clip-path: polygon(0 0, 100% 0, 50% 100%); }
.geo-timeline__ticks { display: flex; justify-content: space-between; padding-top: 4px; color: #64748b; font-family: monospace; font-size: 11px; }
@media (max-width: 960px) { .map { aspect-ratio: 2 / 1; } }
</style>
