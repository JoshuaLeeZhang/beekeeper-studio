<template>
  <div
    class="json-viewer"
    :class="{ 'empty': empty }"
    ref="sidebar"
    v-show="!hidden"
  >
    <div class="header">
      <div
        class="header-group"
        v-show="!empty"
      >
        <div class="filter-wrap">
          <input
            class="form-control"
            type="text"
            placeholder="Filter keys by text or /regex/"
            v-model="debouncedFilter"
          >
          <button
            type="button"
            class="clear btn-link"
            @click="setFilter('')"
            v-if="filter"
          >
            <i class="material-icons">cancel</i>
          </button>
        </div>
        <x-button
          class="menu-btn btn btn-fab"
          tabindex="0"
        >
          <i class="material-icons">more_vert</i>
          <x-menu style="--target-align:right;">
            <x-menuitem
              v-for="option in menuOptions"
              :key="option.name"
              :toggled="option.checked"
              :togglable="typeof option.checked !== 'undefined'"
              @click.prevent="option.handler"
            >
              <x-label>{{ option.name }}</x-label>
            </x-menuitem>
          </x-menu>
        </x-button>
      </div>
    </div>
    <div class="text-editor-wrapper">
      <text-editor
        language-id="json"
        :fold-all="foldAll"
        :unfold-all="unfoldAll"
        :value="cachedJson"
        :force-initialize="reinitializeTextEditor + (reinitialize ?? 0)"
        :markers="markers"
        :replace-extensions="replaceExtensions"
        :line-wrapping="wrapText"
        :line-gutters="lineGutters"
        :line-numbers="false"
        :fold-gutters="true"
      />
    </div>
    <div class="empty-text">
      Open a table to view its data
    </div>
  </div>
</template>

<script lang="ts">
/**
 * hidden:  it's recommended to use `hidden` prop instead of v-show so that
 *          the text editor can be reinitialized.
 * dataId:  use this to update the component with new data.
 */
import Vue from "vue";
import TextEditor from "@beekeeperstudio/ui-kit/vue/text-editor"
import {
  ExpandablePath,
  findKeyPosition,
  createExpandableTextDecoration,
  createTruncatableTextDecoration,
  deepFilterObjectProps,
  cloneForJsonDisplay,
  getTruncatablePaths,
  valueMarkerRange,
  JsonSourcePointers,
} from "@/lib/data/jsonViewer";
import { mapGetters } from "vuex";
import { EditorMarker, LineGutter } from "@beekeeperstudio/ui-kit";
import { persistJsonFold } from "@/lib/editor/extensions/persistJsonFold";
import { partialReadonly } from "@/lib/editor/extensions/partialReadOnly";
import rawLog from "@bksLogger";
import _ from "lodash";
import JsonSourceMap from "json-source-map";
import JsonPointer from "json-pointer";
import { monokaiInit } from "@uiw/codemirror-theme-monokai";

const log = rawLog.scope("json-viewer");

export default Vue.extend({
  components: { TextEditor },
  props: {
    value: {
      type: [Object, Array],
      default: () => ({})
    },
    hidden: {
      type: Boolean,
      default: false
    },
    expandablePaths: {
      type: Array,
      default: () => []
    },
    editablePaths: {
      type: Array,
      default: () => []
    },
    dataId: [String, Number],
    title: String,
    reinitialize: null,
    signs: {
      type: Object,
      default: () => ({})
    },
    binaryEncoding: String,
    filter: {
      type: String,
      default: ""
    },
  },
  data() {
    return {
      reinitializeTextEditor: 0,
      foldAll: 0,
      unfoldAll: 0,
      restoredTruncatedPaths: [],
      editableRangeErrors: [],
      wrapText: false,
      skipFoldPersist: false,
      persistJsonFold: persistJsonFold(),
      partialReadonly: partialReadonly(),
      cacheKey: "",
      cachedJson: "",
      cachedPointers: {} as JsonSourcePointers,
      cachedTruncatablePaths: [] as string[],
      cachedEditableRanges: [] as Array<{
        id: string;
        from: { line: number; ch: number };
        to: { line: number; ch: number };
      }>,
      sourceMapGeneration: 0,
    };
  },
  watch: {
    hidden() {
      if (!this.hidden) {
        this.reinitializeTextEditor++;
        this.refreshJsonCache();
      }
    },
    dataId() {
      this.restoredTruncatedPaths = [];
      this.skipFoldPersist = true;
      this.refreshJsonCache();
      if (this.expandFKDetailsByDefault) {
        this.expandablePaths.forEach((expandablePath: ExpandablePath) => {
          if (expandablePath.path.length === 1) {
            this.expandPath(expandablePath);
          }
        });
      }
    },
    filter() {
      this.refreshJsonCache();
    },
    value() {
      this.refreshJsonCache();
    },
    restoredTruncatedPaths() {
      this.refreshJsonCache();
    },
    editablePaths() {
      this.scheduleSourceMapBuild();
    },
    async cachedJson(newText, oldText) {
      if (this.skipFoldPersist) {
        this.skipFoldPersist = false;
        return;
      }
      if (!oldText || newText === oldText) {
        return;
      }
      this.persistJsonFold.save()
      await this.$nextTick()
      setTimeout(() => this.persistJsonFold.apply())
    },
    cachedEditableRanges(ranges) {
      this.partialReadonly.setEditableRanges(ranges);
    },
  },
  computed: {
    sidebarTitle() {
      return this.title ?? "JSON Row Viewer"
    },
    empty() {
      return _.isEmpty(this.value);
    },
    debouncedFilter: {
      get() {
        return this.filter;
      },
      set: _.debounce(function (value) {
        this.setFilter(value);
      }, 500),
    },
    truncatedPaths() {
      return _.difference(this.cachedTruncatablePaths, this.restoredTruncatedPaths)
    },
    markers() {
      const markers: EditorMarker[] = [];
      _.forEach(this.expandablePaths, (expandablePath: ExpandablePath) => {
        try {
          const pointer = JsonPointer.compile(expandablePath.path.map(String));
          const range = valueMarkerRange(
            this.cachedJson,
            expandablePath.path,
            this.cachedPointers,
            pointer
          );
          if (!range) {
            return;
          }
          markers.push({
            type: "custom",
            from: { line: range.line, ch: range.from },
            to: { line: range.line, ch: range.to },
            decoration: createExpandableTextDecoration(range.value, () => {
              this.expandPath(expandablePath);
            }),
          });
        } catch (e) {
          log.warn("Failed to mark expandable path", expandablePath);
          log.warn(e);
        }
      });
      _.forEach(this.truncatedPaths, (path) => {
        if (this.expandablePaths.some((entry: ExpandablePath) => entry.path.join(".") === path)) {
          return;
        }
        try {
          const segments = path.split(".");
          const pointer = JsonPointer.compile(segments);
          const range = valueMarkerRange(
            this.cachedJson,
            segments,
            this.cachedPointers,
            pointer
          );
          if (!range) {
            return;
          }
          markers.push({
            type: "custom",
            from: { line: range.line, ch: range.from },
            to: { line: range.line, ch: range.to },
            decoration: createTruncatableTextDecoration(range.value, () => {
              this.restoredTruncatedPaths.push(path);
            }),
          });
        } catch (e) {
          log.warn("Failed to mark truncated path", path);
          log.warn(e);
        }
      })
      _.forEach(this.editableRangeErrors, ({ error, from, to }) => {
        markers.push({
          type: "error",
          from,
          to,
          message: error.message,
        });
      })
      return markers;
    },
    lineGutters() {
      const lineGutters: LineGutter[] = []
      _.forEach(this.signs, (_i, key) => {
        const type = this.signs[key]
        const pointer = JsonPointer.compile([key]);
        const range = valueMarkerRange(
          this.cachedJson,
          [key],
          this.cachedPointers,
          pointer
        );
        const line = range?.line ?? findKeyPosition(this.cachedJson, [key]);
        if (line === -1) {
          log.warn(`Failed to sign key \`${key}\`. \`${key}\` is not found.`)
          return
        }
        lineGutters.push({ line, type });
      })
      return lineGutters;
    },
    editableRanges() {
      return this.cachedEditableRanges;
    },
    menuOptions() {
      return [
        {
          name: "Copy Visible",
          handler: () => {
            this.$native.clipboard.writeText(this.cachedJson);
          },
        },
        {
          name: "Collapse all",
          handler: () => {
            this.foldAll++;
          },
        },
        {
          name: "Expand all",
          handler: () => {
            this.unfoldAll++;
          },
        },
        {
          name: "Always Expand Foreign Keys",
          handler: () => {
            this.$store.dispatch("toggleExpandFKDetailsByDefault");
          },
          checked: this.expandFKDetailsByDefault,
        },
        {
          name: "Wrap Text",
          handler: () => {
            this.wrapText = !this.wrapText
          },
          checked: this.wrapText,
        },

      ]
    },
    ...mapGetters(["expandFKDetailsByDefault"]),
  },
  methods: {
    buildCacheKey() {
      return [
        this.dataId,
        this.filter,
        this.restoredTruncatedPaths.join(","),
        this.binaryEncoding,
        _.isEmpty(this.editablePaths) ? "0" : "1",
      ].join("|");
    },
    buildFilteredValue() {
      const truncatedPaths = _.difference(
        getTruncatablePaths(this.value as Record<string, unknown>),
        this.restoredTruncatedPaths
      );
      const processed = cloneForJsonDisplay(this.value as Record<string, unknown>, {
        binaryEncoding: this.binaryEncoding as "hex" | "base64" | undefined,
        truncatedPaths,
      });
      if (!this.filter) {
        return processed;
      }
      return deepFilterObjectProps(processed, this.filter);
    },
    refreshJsonCache() {
      if (this.hidden || this.empty) {
        this.cacheKey = "";
        this.cachedJson = "";
        this.cachedPointers = {};
        this.cachedTruncatablePaths = [];
        this.cachedEditableRanges = [];
        this.sourceMapGeneration++;
        return;
      }

      const cacheKey = this.buildCacheKey();
      if (cacheKey === this.cacheKey) {
        return;
      }

      this.cacheKey = cacheKey;
      this.cachedTruncatablePaths = getTruncatablePaths(this.value as Record<string, unknown>);

      const filteredValue = this.buildFilteredValue();
      this.cachedJson = JSON.stringify(filteredValue, null, 2);
      this.cachedPointers = {};
      this.cachedEditableRanges = [];
      this.scheduleSourceMapBuild(filteredValue);
    },
    scheduleSourceMapBuild(filteredValue = this.buildFilteredValue()) {
      this.sourceMapGeneration++;
      const generation = this.sourceMapGeneration;

      if (_.isEmpty(this.editablePaths)) {
        return;
      }

      const build = () => {
        if (generation !== this.sourceMapGeneration) {
          return;
        }

        try {
          const sourceMap = JsonSourceMap.stringify(filteredValue, null, 2);
          if (generation !== this.sourceMapGeneration) {
            return;
          }
          this.cachedPointers = sourceMap.pointers;
          this.cachedEditableRanges = this.buildEditableRanges(sourceMap.pointers);
        } catch (error) {
          log.warn("Failed to build JSON source map for editable ranges", error);
        }
      };

      if (typeof requestIdleCallback !== "undefined") {
        requestIdleCallback(build);
      } else {
        setTimeout(build, 0);
      }
    },
    buildEditableRanges(pointers: JsonSourcePointers = this.cachedPointers) {
      if (_.isEmpty(this.editablePaths) || _.isEmpty(pointers)) {
        return [];
      }

      const ranges = [];

      this.editablePaths.forEach((path: string) => {
        const pointer = JsonPointer.compile(path.split("."));
        const position = pointers[pointer];

        if (!position) {
          log.warn(`Unable to find editable path \`${path}\` in value object.`);
          return;
        }

        ranges.push({
          id: path,
          from: { line: position.value.line, ch: position.value.column },
          to: { line: position.valueEnd.line, ch: position.valueEnd.column },
        });
      });

      return ranges;
    },
    expandPath(path: ExpandablePath) {
      this.$emit("expandPath", path);
    },
    setFilter(filter: string) {
      this.$emit("bks-filter-change", { filter });
    },
    replaceExtensions(extensions) {
      return [
        extensions,
        monokaiInit({
          settings: {
            selection: "",
            selectionMatch: "",
          },
        }),
        this.persistJsonFold.extensions,
        this.partialReadonly.extensions(this.editableRanges),
      ]
    },
    handleEditableRangeChange: _.debounce(function (range, value) {
      this.editableRangeErrors = []
      try {
        const parsed = JSON.parse(value)
        this.$emit("bks-json-value-change", {key: range.id, value: parsed});
      } catch (error) {
        this.editableRangeErrors.push({ id: range.id, error, from: range.from, to: range.to })
      }
    }, 250),
  },
  mounted() {
    this.partialReadonly.addListener("change", this.handleEditableRangeChange)
    this.refreshJsonCache();
  },
  beforeDestroy() {
    this.partialReadonly.removeListener("change", this.handleEditableRangeChange)
  },
});
</script>
