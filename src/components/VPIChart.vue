<template>
  <div>
    <slot name="header" />
    <b-form inline>
      <b-form-group
        id="vpi-display-mode-group"
        :label="t('Variable:')"
        label-for="vpi-display-mode"
      >
        <b-form-select
          id="vpi-display-mode"
          v-model="selectedMode"
          size="sm"
          class="mx-3"
          :options="availableModesTranslated"
          value-field="propertyName"
          text-field="descriptionCurrentLanguage"
        />
      </b-form-group>
    </b-form>

    <div id="ignore-mouse-events" style="pointer-events: none" />

    <svg
      id="new-vpi-chart"
      :width="styleConfig.width"
      :height="styleConfig.height"
      class="d-block mx-auto"
    >
      <text v-if="noData" x="50%" y="50%" text-anchor="middle" fill="#6c757d">{{ t("No data found") }}</text>

      <g :transform="`translate(${margin.left}, ${margin.top})`">
        <!-- X axis -->
        <g :transform="`translate(0, ${innerHeight})`">
          <slot name="in-x-axis-group" />
          <g
            v-xaxis="{
              scale: xScale,
              timezone: showTimeAs,
              axisTimeFormat: styleConfig.axisTimeFormat,
            }"
          />
        </g>

        <!-- Y axis -->
        <g v-yaxis="{ scale: yScale }" />

        <!-- Y axis legend -->
        <text
          text-anchor="middle"
          transform="rotate(-90)"
          :y="-margin.left + 20"
          :x="-margin.top - 110"
        >
          {{ selectedModeLabel }}
        </text>

        <!-- tooltip -->
        <template v-if="styleConfig.showTooltip">
          <rect
            style="visibility: hidden"
            pointer-events="all"
            :width="innerWidth"
            :height="innerHeight"
            @mousemove="mouseMove"
            @mouseenter="mouseEnter"
            @mouseleave="mouseLeave"
          />
          <circle
            v-show="tooltipVisible"
            id="tooltipCircle"
            style="pointer-events: none"
            :cx="closestMomentXPosition"
            :cy="YPositionAtTimeX"
            r="4"
            :style="`fill: ${styleConfig.strokeColor}`"
          />

          <b-popover
            container="ignore-mouse-events"
            :show.sync="tooltipVisible"
            target="tooltipCircle"
            placement="top"
            :no-fade="true"
            :delay="0"
          >
            <!-- eslint-disable-next-line vue/valid-v-slot -->
            <template #title>{{ formattedMomentAtTimeX }}</template>
            <div>
              {{ selectedModeLabel }}:
              {{ selectedValAtTimeX | rounddecimals(selectedModePrecision) }}
            </div>
          </b-popover>
        </template>

        <!-- finally, the chart line -->
        <path
          style="pointer-events: none"
          :stroke="styleConfig.strokeColor"
          :fill="styleConfig.fillColor"
          :stroke-width="styleConfig.strokeWidth"
          :d="pathData"
        />

        <daily-lines v-if="!noData" :days="daysCovered" :height="innerHeight" />
      </g>
    </svg>
  </div>
</template>

<script lang="ts">
/* eslint-disable vue/require-default-prop */
import Vue from "vue";
import * as d3 from "d3";
import moment from "moment-timezone";

import DailyLines from "@/components/DailyLines.vue";

import helpers from "@/helpers";

import { DayData, VPIEntry, IntegratedPropertyName, LangCode, MultilanguageStringContainer } from "@/CrowTypes";

import TWEEN from "@tweenjs/tween.js";
import { UserChoicesStoreModule } from "@/store/UserChoicesStore";

const MinRTRValueDisplay = 50;  // If the maximum MTR is small, we return 50 so a small peak on a very calm day doesn't seem huge on the chart

type NullableNumber = number | null;
type NullableVPIEntry = VPIEntry | null;

interface DisplayMode {
  propertyName: IntegratedPropertyName; // the name of the property (on vpiData[].integratedProfiles) where data can be found. Can be used as an ID
  variableName: string; // appears as legend of the Y axis and in popover
  variableDescriptionId: string; // to appear in <select> (after i18n mechanism)
  yMaxValComputedName: "maxMTRWithMinimum" | "maxRTR" | "maxVID" | "maxVIR"; // the name of a computed property to get the max value for the Y Axis
}

interface DisplayModeTranslated extends DisplayMode {
  descriptionCurrentLanguage: string
}

interface VPIEntryForPath {
  // "prepared" VPI data to draw path, derived from VPIEntry. Changes:
  // - time data stored as a timestamp (no further conversion needed)
  // - only single value to display (allow to animate when the user switch between MTR and VID, for example)
  timestamp: number;
  val: number; // scaled value (Y coordinate)
  sourceVal: number; // source data (no scaling - NaN if missing data)
}

export default Vue.extend({
  name: "VPIChart",
  components: {
    DailyLines,
  },
  directives: {
    yaxis(el, binding): void {
      const scaleFunction = binding.value.scale;

      const d3Axis = d3
        .axisLeft<number>(scaleFunction)
        .tickFormat(d3.format("d"))
        .tickSizeOuter(0); // And we want to hide the last tick line

      d3Axis(d3.select((el as unknown) as SVGGElement)); // TODO: TS: There's probably a better solution than this double casting
    },
    xaxis(el, binding): void {
      // TODO: code copy/pasted from VPChart. Possible to factorize (without mixins)? Or isn't it worth it?
      const scaleFunction = binding.value.scale;
      const showTimeAs = binding.value.timezone;
      const axisTimeFormat = binding.value.axisTimeFormat;

      const d3Axis = d3
        .axisBottom<number>(scaleFunction)
        .ticks(7)
        .tickSize(15)
        .tickFormat((d) => {
          return helpers.formatTimestamp(d, showTimeAs, axisTimeFormat);
        });

      d3Axis(d3.select((el as unknown) as SVGGElement)); // TODO: TS: There's probably a better solution than this double casting
    },
  },
  filters: {
    rounddecimals: function (num: number, precision: number): string {
      const multiplier = Math.pow(10, precision);
      return (Math.round(num * multiplier) / multiplier).toFixed(precision);
    },
  },
  props: {
    // eslint-disable-next-line vue/require-default-prop
    vpiData: Array as () => VPIEntry[],
    styleConfig: Object,
    showTimeAs: String, // "UTC" or a TZ database entry (such as "Europe/Brussels")
    appTemporalResolution: Number,
    mode: String as () => IntegratedPropertyName,
    loading: Boolean
  },
  data: function () {
    return {
      vpiDataForPath: [] as VPIEntryForPath[],
      selectedMode: this.mode,
      availableModes: [
        {
          variableName: "Migration traffic rate (MTR)",
          variableDescriptionId: "MTR description",
          propertyName: "mtr",
          yMaxValComputedName: "maxMTRWithMinimum",
        },
        {
          variableName: "Reflectivity traffic rate (RTR)",
          variableDescriptionId: "RTR description",
          propertyName: "rtr",
          yMaxValComputedName: "maxRTR",
        },
        {
          variableName: "Vertically integrated density (VID)",
          variableDescriptionId: "VID description",
          propertyName: "vid",
          yMaxValComputedName: "maxVID",
        },
        {
          variableName: "Vertically integrated reflectivity (VIR)",
          variableDescriptionId: "VIR description",
          propertyName: "vir",
          yMaxValComputedName: "maxVIR",
        },
      ] as DisplayMode[],

      texts: {
        "Variable:": {
          en: "Variable:",
          fr: "Variable :",
          nl: "Variabele"
        },
        "MTR description": {
          en: "MTR: hourly number of birds flying over a 1 km transect",
          fr: "MTR: nombre horaire d'oiseaux survolant un transect de 1 km",
          nl: "MTR: aantal vogels dat per uur over een transect van 1 km vliegt"
        },
        "RTR description": {
          en: "RTR: hourly reflected bird surface (cm²) passing over a 1 km transect",
          fr: "RTR: surface horaire réfléchie par les oiseaux (cm²) passant un transect de 1 km",
          nl: "RTR: gereflecteerde vogeloppervlakte (cm²) dat per uur over een transect van 1 km passeert"
        },
        "VID description": {
          en: "VID: number of birds per km²",
          fr: "VID: nombre d'oiseaux par km²",
          nl: "VID: aantal vogels per km²"
        },
        "VIR description": {
          en: "VIR: reflected bird surface (cm²) per km²",
          fr: "VIR: surface réfléchie par les oiseaux (cm²) par km²",
          nl: "VIR: gereflecteerde vogeloppervlakte (cm²) per km²"
        },
        "No data found": {
          en: "No data found",
          fr: "Données introuvables",
          nl: "Geen data gevonden"
        }
      } as MultilanguageStringContainer,

      margin: this.styleConfig.margin,

      tooltipVisible: false,

      // For tooltip: those will be null if the mouse is *not* over the chart
      // See also computed properties: selectedValAtTimeX and YPositionAtTimeX
      mouseXPosition: null as NullableNumber, // in pixels, 0 is left border of the graph
      VPIEntryAtTimeX: null as NullableVPIEntry,

      momentBisector: d3.bisector(function (d: VPIEntry) {
        return d.moment.valueOf();
      }).left,

      timestampBisector: d3.bisector(function (d: VPIEntryForPath) {
        return d.timestamp;
      }).left,

      innerWidth:
        this.styleConfig.width -
        this.styleConfig.margin.left -
        this.styleConfig.margin.right,
      innerHeight:
        this.styleConfig.height -
        this.styleConfig.margin.top -
        this.styleConfig.margin.bottom,
    };
  },
  computed: {
    availableModesTranslated(): DisplayModeTranslated[] {
      return this.availableModes.map(mode => ({...mode, descriptionCurrentLanguage: this.t(mode.variableDescriptionId)}))
    },
    selectedLanguageCode(): LangCode {
      return UserChoicesStoreModule.selectedLanguageCode;
    },
    selectedModePrecision(): number {
      switch (this.selectedMode) {
        case "vid":
          return 1; // all other modes currently don't need to display decimal digits
        default:
          return 0;
      }
    },
    daysCovered: function (): DayData[] {
      // Find the day covered by each entry in vpiData
      const coveredDays = this.vpiData.map((vpiEntry) => {
        return vpiEntry.moment.clone().tz(this.showTimeAs).startOf("day");
      });

      // Remove duplicates
      const comparisonValues = coveredDays.map((v) => v.valueOf());
      const uniqueCoveredDays = coveredDays.filter(
        (v, i) => comparisonValues.indexOf(v.valueOf()) == i
      );

      return uniqueCoveredDays.map((mom) => {
        return {
          moment: mom,
          xPositionAtMidnight: this.xScale(mom.valueOf()),
          dayLabel: mom.format("MMM DD"),
        };
      });
    },
    formattedMomentAtTimeX: function (): string {
      if (this.VPIEntryAtTimeX) {
        return helpers.formatMoment(
          this.VPIEntryAtTimeX.moment,
          this.showTimeAs,
          this.styleConfig.tooltipTimeFormat
        );
      }
      return "";
    },
    YPositionAtTimeX: function (): number {
      if (this.selectedValAtTimeX) {
        return this.yScale(this.selectedValAtTimeX);
      }
      return this.innerHeight;
    },
    selectedValAtTimeX: function (): number | null {
      if (this.VPIEntryAtTimeX) {
        return this.VPIEntryAtTimeX.integratedProfiles[
          this.selectedModePropertyName
        ];
      }
      return null;
    },
    closestMomentXPosition: function (): number | null {
      if (this.VPIEntryAtTimeX) {
        return this.xScale(this.VPIEntryAtTimeX.moment.valueOf());
      }
      return null;
    },
    maxVID: function (): number {
      const max = d3.max(this.vpiData, function (d) {
        return d.integratedProfiles.vid;
      });
      return max || 0;
    },
    maxVIR: function (): number {
      const max = d3.max(this.vpiData, function (d) {
        return d.integratedProfiles.vir;
      });
      return max || 0;
    },
    maxMTR: function (): number {
      const max = d3.max(this.vpiData, function (d) {
        return d.integratedProfiles.mtr;
      });
      return max || 0;
    },
    maxRTR: function (): number {
      const max = d3.max(this.vpiData, function (d) {
        return d.integratedProfiles.rtr;
      });
      return max || 0;
    },
    maxMTRWithMinimum: function (): number {
      if (this.maxMTR < MinRTRValueDisplay) {
        return MinRTRValueDisplay;
      } else {
        return this.maxMTR;
      }
    },
    noData: function (): boolean {
      const emptyMaxValue = this.selectedModeObject.yMaxValComputedName === 'maxMTRWithMinimum' ? MinRTRValueDisplay : 0;
      return (this.yMaxVal === emptyMaxValue) && !this.loading;
    },
    yMaxVal: function (): number {
      return this[this.selectedModeObject.yMaxValComputedName];
    },
    selectedModePropertyName: function (): IntegratedPropertyName {
      return this.selectedModeObject.propertyName;
    },
    selectedModeLabel: function (): string {
      return this.selectedModeObject.variableName;
    },
    selectedModeObject: function (): DisplayMode {
      const found = this.availableModes.find(
        (d) => d.propertyName == this.selectedMode
      );
      return found || this.availableModes[0]; // Default: first entry
    },
    minMoment: function (): moment.Moment {
      const foundMoment = d3.min(this.vpiData, function (d) {
        return d.moment;
      });
      return foundMoment || moment();
    },
    maxMoment: function (): moment.Moment {
      const foundMoment = d3.max(this.vpiData, function (d) {
        return d.moment;
      });
      return foundMoment || moment();
    },
    maxMomentPlusOne: function (): moment.Moment {
      // TODO: duplicate code in other charts ! Mixin? Helper?
      return this.maxMoment.clone().add(this.appTemporalResolution, "seconds");
    },
    xScale: function (): d3.ScaleTime<number, number> {
      return d3
        .scaleTime()
        .domain([this.minMoment.valueOf(), this.maxMomentPlusOne.valueOf()])
        .range([0, this.innerWidth]);
    },
    yScale: function (): d3.ScaleLinear<number, number> {
      return d3
        .scaleLinear()
        .range([this.innerHeight, 0])
        .domain([0, this.yMaxVal]);
    },
    pathData: function (): string | null {
      const path = d3
        .area<VPIEntryForPath>()
        .defined((vpiEntryFP) => {
          return !isNaN(vpiEntryFP.sourceVal);
        })
        .x((vpiEntryFP) => {
          return this.xScale(vpiEntryFP.timestamp);
        })
        .y0((vpiEntryFP) => {
          return vpiEntryFP.val;
        })
        .y1(this.innerHeight);

      return path(this.vpiDataForPath);
    },
  },
  watch: {
    selectedMode: {
      immediate: true,
      handler: function (newMode): void {
        this.syncVPIDataForPath();
        this.animate();
        this.$emit("mode-changed", newMode);
      },
    },
    vpiData: {
      immediate: true,
      handler: function (): void {
        this.syncVPIDataForPath();
        this.animate();
      },
    },
  },
  methods: {
    t(stringId: string) {
      return helpers.translateString(stringId, this.selectedLanguageCode, this.texts);
    },
    animate(): void {
      if (TWEEN.update(TWEEN.now())) {
        requestAnimationFrame(this.animate);
      }
    },
    syncVPIDataForPath(): void {
      // We smoothly update each entry in vpiDataForPath, based on selectedModePropertyName and vpiData
      // 1. Remove outdated entries first, so the index don't change later on (tween's callbacks, ...)
      // 2. Add new entries, if necessary (when populating vpiData for example)
      // 3. Update .val, based on selectedModePropertyName

      // TODO: refactor this method.

      // 1. Remove oudated entries
      const timestampsInVPI = [] as number[];
      for (const entry of this.vpiData) {
        timestampsInVPI.push(entry.moment.valueOf());
      }
      const indexOfEntriesToRemove = [] as number[];
      this.vpiDataForPath.forEach((entryForPath, index) => {
        if (!timestampsInVPI.includes(entryForPath.timestamp)) {
          indexOfEntriesToRemove.push(index);
        }
      });

      // Removal happens while iterating BACKWARDS so indexes stay valid all along the loop
      for (let i = indexOfEntriesToRemove.length - 1; i >= 0; --i) {
        this.vpiDataForPath.splice(indexOfEntriesToRemove[i], 1);
      }

      for (const entry of this.vpiData) {
        const timestamp = entry.moment.valueOf();
        const foundIndex = this.vpiDataForPath.findIndex((element) => {
          return element.timestamp === timestamp;
        });

        const rawValue =
          entry.integratedProfiles[this.selectedModePropertyName];
        const newScaledValue = this.yScale(isNaN(rawValue) ? 0 : rawValue);

        if (foundIndex == -1) {
          // 2. Add new data
          const insertIndex = this.timestampBisector(
            this.vpiDataForPath,
            timestamp
          );

          const newEntry = {
            timestamp: timestamp,
            val: newScaledValue,
            sourceVal: rawValue
          };

          this.vpiDataForPath.splice(insertIndex, 0, newEntry);
        } else {
          // 3. Update existing data (and tween it)
          const cloneElem = { ...this.vpiDataForPath[foundIndex] };

          new TWEEN.Tween(cloneElem)
            .easing(TWEEN.Easing.Quadratic.Out)
            .to({ val: newScaledValue, sourceVal: rawValue }, 300)
            .onUpdate(() => {
              this.$set(this.vpiDataForPath[foundIndex], "val", cloneElem.val);
              this.$set(this.vpiDataForPath[foundIndex], "sourceVal", cloneElem.sourceVal);
            })
            .start(TWEEN.now()); // Start the tween immediately.
        }
      }
    },
    mouseEnter(): void {
      this.tooltipVisible = true;
      this.mouseXPosition = 0;
    },
    mouseLeave(): void {
      this.tooltipVisible = false;
      this.mouseXPosition = null;
      this.VPIEntryAtTimeX = null;
    },
    mouseMove(event: MouseEvent): void {
      if (this.selectedValAtTimeX && !isNaN(this.selectedValAtTimeX)) {
        this.tooltipVisible = true;
      } else {
        this.tooltipVisible = false;
      }

      // When mouse is moved over the chart, updates this.mouseXPosition and this.VPIEntryAtTimeX
      // 1. Get (and save in data) the mouse position
      const target = event.target as HTMLElement;
      const bounds = target.getBoundingClientRect();
      const mouseX = event.clientX - bounds.left;
      this.mouseXPosition = mouseX;

      // 2. Find out (and save in data) the VPI data at the mouse position
      const x0 = this.xScale.invert(this.mouseXPosition);
      const i = this.momentBisector(this.vpiData, x0);
      const d0 = this.vpiData[i - 1];
      const d1 = this.vpiData[i];

      if (d0 && d1) { // Avoid errors at chart boundaries (undefined d0 left, undefined d1 right)
        this.VPIEntryAtTimeX =
          x0.getTime() / 1000 - d0.moment.valueOf() >
            d1.moment.valueOf() - x0.getTime() / 1000
            ? d1
            : d0;
      } else {
        this.VPIEntryAtTimeX = null;
      }

    },
  },
});
</script>
