<template>
  <div
    class="ww-datagrid"
    ref="containerRef"
    :class="{ editing: isEditing, loading: !!content?.loading }"
    :style="cssVars"
  >
    <ag-grid-vue
      :rowData="rowData"
      :columnDefs="columnDefs"
      :initial-state="initialState"
      :defaultColDef="defaultColDef"
      :domLayout="content.layout === 'auto' ? 'autoHeight' : 'normal'"
      :style="style"
      :rowSelection="rowSelection"
      :selection-column-def="{ pinned: true }"
      :theme="theme"
      :getRowId="getRowId"
      :pagination="content.pagination"
      :paginationPageSize="
        forcedPaginationPageSize
          ? 0
          : paginationPageSizeSelector
          ? paginationPageSizeSelector[0]
          : content.paginationPageSize
      "
      :paginationPageSizeSelector="paginationPageSizeSelector"
      :suppressPaginationPanel="!(content?.pagination && content?.showPaginationUI)"
      :suppressMovableColumns="!content.movableColumns"
      :suppressServerSideFullWidthLoadingRow="content?.suppressServerSideFullWidthLoadingRow ?? true"
      :columnHoverHighlight="content.columnHoverHighlight"
      :locale-text="localeText"
      :popupParent="popupParent"
      enableCellTextSelection
      ensureDomOrder
      :row-drag-managed="true"
      @grid-ready="onGridReady"
      @row-selected="onRowSelected"
      @selection-changed="onSelectionChanged"
      @cell-value-changed="onCellValueChanged"
      @filter-changed="onFilterChanged"
      @sort-changed="onSortChanged"
      @row-clicked="onRowClicked"
      @row-drag-end="onRowDragged"
      @row-drag-enter="onRowDragEnter"
      @column-moved="onColumnMoved"
      @column-header-clicked="onHeaderClicked"
      @pagination-changed="onPaginationChanged"
    >
    </ag-grid-vue>
    <div
      v-if="content?.loading"
      class="ww-loader-overlay"
      :style="loaderStyle"
    ></div>
  </div>
</template>

<script>
import {
  shallowRef,
  watchEffect,
  computed,
  inject,
  watch,
  nextTick,
  ref,
} from "vue";
import { AgGridVue } from "ag-grid-vue3";
import {
  AllCommunityModule,
  ModuleRegistry,
  themeQuartz,
} from "ag-grid-community";
import {
  AG_GRID_LOCALE_EN,
  AG_GRID_LOCALE_FR,
  AG_GRID_LOCALE_DE,
  AG_GRID_LOCALE_ES,
  AG_GRID_LOCALE_PT,
} from "@ag-grid-community/locale";
import ActionCellRenderer from "./components/ActionCellRenderer.vue";
import ImageCellRenderer from "./components/ImageCellRenderer.vue";
import WewebCellRenderer from "./components/WewebCellRenderer.vue";
import CustomHeaderComponent from "./components/CustomHeaderComponent.vue";

// TODO: maybe register less modules
// TODO: maybe register modules per grid instead of globally
ModuleRegistry.registerModules([AllCommunityModule]);

export default {
  components: {
    AgGridVue,
    ActionCellRenderer,
    ImageCellRenderer,
    WewebCellRenderer,
    CustomHeaderComponent,
  },
  props: {
    content: {
      type: Object,
      required: true,
    },
    uid: {
      type: String,
      required: true,
    },
    /* wwEditor:start */
    wwEditorState: { type: Object, required: true },
    /* wwEditor:end */
  },
  emits: ["trigger-event", "update:content:effect"],
  setup(props, ctx) {
    const { resolveMappingFormula } = wwLib.wwFormula.useFormula();

    const gridApi = shallowRef(null);
    const containerRef = shallowRef(null);
    const headerHeight = shallowRef(0);
    const { value: selectedRows, setValue: setSelectedRows } =
      wwLib.wwVariable.useComponentVariable({
        uid: props.uid,
        name: "selectedRows",
        type: "array",
        defaultValue: [],
        readonly: true,
      });
    const { value: filterValue, setValue: setFilters } =
      wwLib.wwVariable.useComponentVariable({
        uid: props.uid,
        name: "filters",
        type: "object",
        defaultValue: {},
        readonly: true,
      });
    const { value: sortValue, setValue: setSort } =
      wwLib.wwVariable.useComponentVariable({
        uid: props.uid,
        name: "sort",
        type: "object",
        defaultValue: {},
        readonly: true,
      });
    const { value: columnOrder, setValue: setColumnOrder } =
      wwLib.wwVariable.useComponentVariable({
        uid: props.uid,
        name: "columnOrder",
        type: "array",
        defaultValue: [],
        readonly: true,
      });

    const onGridReady = (params) => {
      gridApi.value = params.api;
      // Measure header height to position loader below headers
      setTimeout(() => {
        try {
          const rootEl = containerRef?.value;
          const headerEl = rootEl?.querySelector?.(".ag-header");
          const rect = headerEl?.getBoundingClientRect?.();
          headerHeight.value = Math.max(0, Math.floor(rect?.height || 0));
        } catch (e) {
          headerHeight.value = 0;
        }
      }, 0);
      const columns = params.api.getAllGridColumns();
      setColumnOrder(columns.map((col) => col.getColId()));
    };

    let initialFilter = "";
    let initialSort = "";

    watchEffect(() => {
      // Both initial filters and sort should be set here to avoid conflicts with column state application
      // We keep track of previous values to avoid reinitializing one when only the other changes
      if (!gridApi.value) return;
      if (
        props.content.initialFilters &&
        initialFilter !== JSON.stringify(props.content.initialFilters)
      ) {
        gridApi.value.setFilterModel(props.content.initialFilters);
        initialFilter = JSON.stringify(props.content.initialFilters);
      }
      if (
        props.content.initialSort &&
        initialSort !== JSON.stringify(props.content.initialSort)
      ) {
        gridApi.value.applyColumnState({
          state: props.content.initialSort || [],
          defaultState: { sort: null },
        });
        initialSort = JSON.stringify(props.content.initialSort);
      }
    });

    watchEffect(() => {
      if (!gridApi.value) return;
      if (props.content.initialColumnsOrder) {
        gridApi.value.applyColumnState({
          state: props.content.initialColumnsOrder.map((colId) => ({ colId })),
          applyOrder: true,
        });
      }
    });

    const initialState = computed(() => {
      const state = {
        partialColumnState: true,
      };
      if (props.content.initialFilters) {
        state.filter = { filterModel: props.content.initialFilters };
      }
      if (props.content.initialSort) {
        state.sort = { sortModel: props.content.initialSort };
      }
      if (props.content.initialColumnsOrder) {
        state.columnOrder = {
          orderedColIds: props.content.initialColumnsOrder,
        };
      }
      return state;
    });

    const onRowSelected = (event) => {
      const name = event.node.isSelected() ? "rowSelected" : "rowDeselected";
      ctx.emit("trigger-event", {
        name,
        event: { row: event.data },
      });
    };

    const onRowDragged = (event) => {
      const rows = [];
      event.api.forEachNode((node) => {
        rows.push(node.data);
      });
      ctx.emit("trigger-event", {
        name: "rowDragged",
        event: {
          row: event.node.data,
          id: event.node.id,
          targetIndex: event.overIndex,
          rows,
        },
      });
    };

    const onRowDragEnter = (event) => {
      ctx.emit("trigger-event", {
        name: "rowDragStart",
        event: {
          row: event.node.data,
          id: event.node.id,
        },
      });
    };

    const onSelectionChanged = (event) => {
      if (!gridApi.value) return;
      const selected = gridApi.value.getSelectedRows() || [];
      setSelectedRows(selected);
    };

    const onFilterChanged = (event) => {
      if (!gridApi.value) return;
      const filterModel = gridApi.value.getFilterModel();
      if (
        JSON.stringify(filterModel || {}) !==
        JSON.stringify(filterValue.value || {})
      ) {
        setFilters(filterModel);
        ctx.emit("trigger-event", {
          name: "filterChanged",
          event: filterModel,
        });
      }
    };

    const onSortChanged = (event) => {
      if (!gridApi.value) return;
      const state = gridApi.value.getState();
      if (
        JSON.stringify(state.sort?.sortModel || []) !==
        JSON.stringify(sortValue.value || [])
      ) {
        setSort(state.sort?.sortModel || []);
        ctx.emit("trigger-event", {
          name: "sortChanged",
          event: state.sort?.sortModel || [],
        });
      }
    };

    const onColumnMoved = (event) => {
      if (!event.finished || event.source !== "uiColumnMoved") return;
      const columns = event.api.getAllGridColumns();
      setColumnOrder(columns.map((col) => col.getColId()));
      ctx.emit("trigger-event", {
        name: "columnMoved",
        event: {
          toIndex: event.toIndex,
          columnId: event.column.getColId(),
          columnsOrder: columns.map((col) => col.getColId()),
        },
      });
    };

    const onCustomButtonClicked = (event) => {
      ctx.emit("trigger-event", {
        name: "customButtonClicked",
        event: {
          column_id: event.column_id,
          column_def: event.column_def,
          column_provided_def: event.column_provided_def,
          field: event.field,
          displayName: event.displayName,
        },
      });
    };

    const onPaginationChanged = () => {
      if (!gridApi.value) return;
      const api = gridApi.value;
      const currentPage =
        typeof api.paginationGetCurrentPage === "function"
          ? api.paginationGetCurrentPage()
          : 0;
      const pageSize =
        typeof api.paginationGetPageSize === "function"
          ? api.paginationGetPageSize()
          : props.content.paginationPageSize || 10;
      const totalRows =
        typeof api.getDisplayedRowCount === "function"
          ? api.getDisplayedRowCount()
          : 0;
      const totalPages = pageSize ? Math.max(1, Math.ceil(totalRows / pageSize)) : 1;
      const firstRowIndex = currentPage * pageSize;
      const lastRowIndex = Math.min(totalRows - 1, firstRowIndex + pageSize - 1);

      ctx.emit("trigger-event", {
        name: "pageChanged",
        event: {
          currentPage,
          pageSize,
          totalPages,
          totalRows,
          firstRowIndex,
          lastRowIndex,
        },
      });
    };

    /* wwEditor:start */
    const { createElement } = wwLib.useCreateElement();
    /* wwEditor:end */

    // Hack to force pagination page size update when changing pagination selector mode
    const forcedPaginationPageSize = ref(false);
    watch(
      () => props.content.hasPaginationSelector,
      (newVal, oldVal) => {
        if (oldVal === "multiple" && newVal !== "multiple") {
          forcedPaginationPageSize.value = true;
          nextTick().then(() => {
            forcedPaginationPageSize.value = false;
          });
        }
      }
    );

    const rowData = computed(() => {
      const data = wwLib.wwUtils.getDataFromCollection(props.content.rowData);
      return Array.isArray(data) ? data ?? [] : [];
    });

    function refreshData() {
      nextTick(() => {
        gridApi.value?.refreshCells()
      });
    }

    return {
      resolveMappingFormula,
      onGridReady,
      onRowSelected,
      onSelectionChanged,
      gridApi,
      onFilterChanged,
      onSortChanged,
      onCustomButtonClicked,
      onPaginationChanged,
      containerRef,
      loaderStyle: computed(() => ({
        "--ww-loader-top": `${headerHeight.value}px`,
      })),
      popupParent: computed(() => {
        const doc = wwLib?.getFrontDocument?.();
        return doc?.body;
      }),
      // Actions
      goToPage: (page) => {
        if (!gridApi.value || !props.content.pagination) return;
        const api = gridApi.value;
        const pageNum = Number(page);
        if (!Number.isFinite(pageNum)) return;
        const zeroBased = Math.max(0, Math.floor(pageNum - 1));
        const totalPages =
          typeof api.paginationGetTotalPages === "function"
            ? api.paginationGetTotalPages()
            : undefined;
        const targetPage =
          typeof totalPages === "number" && isFinite(totalPages)
            ? Math.min(zeroBased, Math.max(0, totalPages - 1))
            : zeroBased;
        if (typeof api.paginationGoToPage === "function") {
          api.paginationGoToPage(targetPage);
        }
      },
      nextPage: () => {
        if (!gridApi.value || !props.content.pagination) return;
        const api = gridApi.value;
        if (typeof api.paginationGoToNextPage === "function") {
          api.paginationGoToNextPage();
        } else if (typeof api.paginationGetCurrentPage === "function") {
          const current = api.paginationGetCurrentPage() || 0;
          if (typeof api.paginationGoToPage === "function") {
            api.paginationGoToPage(current + 1);
          }
        }
      },
      prevPage: () => {
        if (!gridApi.value || !props.content.pagination) return;
        const api = gridApi.value;
        if (typeof api.paginationGoToPreviousPage === "function") {
          api.paginationGoToPreviousPage();
        } else if (typeof api.paginationGetCurrentPage === "function") {
          const current = api.paginationGetCurrentPage() || 0;
          if (typeof api.paginationGoToPage === "function") {
            api.paginationGoToPage(Math.max(0, current - 1));
          }
        }
      },
      setPageSize: (pageSize) => {
        if (!gridApi.value || !props.content.pagination) return;
        const api = gridApi.value;
        const size = Number(pageSize);
        if (!Number.isFinite(size) || size <= 0) return;
        if (typeof api.paginationSetPageSize === "function") {
          api.paginationSetPageSize(size);
        }
      },
      localeText: computed(() => {
        switch (props.content.lang) {
          case "fr":
            return AG_GRID_LOCALE_FR;
          case "de":
            return AG_GRID_LOCALE_DE;
          case "es":
            return AG_GRID_LOCALE_ES;
          case "pt":
            return AG_GRID_LOCALE_PT;
          case "custom":
            return {
              ...AG_GRID_LOCALE_EN,
              ...(props.content.localeText || {}),
            };
          default:
            AG_GRID_LOCALE_EN;
        }
      }),
      forcedPaginationPageSize,
      onRowDragged,
      onRowDragEnter,
      onColumnMoved,
      initialState,
      refreshData,
      rowData,
      /* wwEditor:start */
      createElement,
      rawContent: inject("componentRawContent", {}),
      /* wwEditor:end */
    };
  },
  computed: {
    defaultColDef() {
      return {
        editable: false,
        resizable: this.content.resizableColumns,
        autoHeaderHeight: this.content.headerHeightMode === "auto",
        wrapHeaderText: this.content.headerHeightMode === "auto",
        cellClass:
          this.content.cellAlignmentMode === "custom"
            ? `-${this.content.cellAlignment || "left"} ||`
            : null,
      };
    },
    columnDefs() {
      const columns = this.content.columns.map((col, index) => {
        const minWidth =
          !col?.minWidth || col?.minWidth === "auto"
            ? null
            : wwLib.wwUtils.getLengthUnit(col?.minWidth)?.[0];
        const maxWidth =
          !col?.maxWidth || col?.maxWidth === "auto"
            ? null
            : wwLib.wwUtils.getLengthUnit(col?.maxWidth)?.[0];
        const width =
          !col?.width || col?.width === "auto" || col?.widthAlgo === "flex"
            ? null
            : wwLib.wwUtils.getLengthUnit(col?.width)?.[0];
        const flex = col?.widthAlgo === "flex" ? col?.flex ?? 1 : null;
        const commonProperties = {
          minWidth,
          maxWidth,
          pinned: col?.pinned === "none" ? false : col?.pinned,
          width,
          flex,
          hide: !!col?.hide,
          headerClass: col.headerAlignment ? `-${col.headerAlignment}` : null,
          ...(this.content.cellAlignmentMode !== "custom"
            ? { cellClass: col.cellAlignment ? `-${col.cellAlignment}` : null }
            : {}),
        };
        const headerProperties = {
          headerComponent: "CustomHeaderComponent",
          headerComponentParams: {
            customButtonTrigger: this.onCustomButtonClicked,
            headerName: col.headerName,
            field: col.field,
            sortable: col.sortable,
            filter: col.filter,
            customButton: col.customButton,
          },
        };
        switch (col?.cellDataType) {
          case "action": {
            return {
              ...commonProperties,
              headerName: col?.headerName,
              cellRenderer: "ActionCellRenderer",
              cellRendererParams: {
                name: col?.actionName,
                label: col?.actionLabel,
                trigger: this.onActionTrigger,
                withFont: !!this.content.actionFont,
              },
              sortable: false,
              filter: false,
              customButton: false,
              colId: col?.actionName,
            };
          }
          case "custom":
            return {
              ...commonProperties,
              ...headerProperties,
              headerName: col?.headerName,
              colId: 'datagrid-header-' + col?.field,
              field: col?.field,
              cellRenderer: "WewebCellRenderer",
              cellRendererParams: {
                containerId: col?.containerId,
              },
              sortable: col?.sortable,
              filter: col?.filter,
              customButton: col?.customButton,
            };
          case "dateString": {
            const parseToMidnight = (value) => {
              if (value == null) return NaN;
              const date = new Date(value);
              if (isNaN(date.getTime())) return NaN;
              const normalized = new Date(
                date.getFullYear(),
                date.getMonth(),
                date.getDate(),
                0,
                0,
                0,
                0
              );
              return normalized.getTime();
            };
            const result = {
              ...commonProperties,
              ...headerProperties,
              headerName: col.headerName,
              colId: 'datagrid-header-' + col.field,
              field: col.field,
              sortable: col.sortable,
              filter: col.filter ? "agDateColumnFilter" : false,
              customButton: col.customButton,
              comparator: (a, b) => {
                const ta = parseToMidnight(a);
                const tb = parseToMidnight(b);
                if (isNaN(ta) && isNaN(tb)) return 0;
                if (isNaN(ta)) return 1;
                if (isNaN(tb)) return -1;
                return ta - tb;
              },
              filterParams: col.filter
                ? {
                    comparator: (filterLocalDateAtMidnight, cellValue) => {
                      const cellTs = parseToMidnight(cellValue);
                      if (isNaN(cellTs)) return -1;
                      const filterTs = filterLocalDateAtMidnight?.getTime?.();
                      if (typeof filterTs !== "number") return -1;
                      if (cellTs === filterTs) return 0;
                      return cellTs < filterTs ? -1 : 1;
                    },
                  }
                : undefined,
            };
            if (col.useCustomLabel) {
              result.valueFormatter = (params) => {
                return this.resolveMappingFormula(
                  col.displayLabelFormula,
                  params.value
                );
              };
            }
            return result;
          }
          case "image": {
            return {
              ...commonProperties,
              headerName: col?.headerName,
              colId: 'datagrid-header-' + col?.field,
              field: col?.field,
              cellRenderer: "ImageCellRenderer",
              cellRendererParams: {
                width: col?.imageWidth,
                height: col?.imageHeight,
              },
            };
          }
          default: {
            const result = {
              ...commonProperties,
              ...headerProperties,
              headerName: col?.headerName,
              colId: 'datagrid-header-' + col?.field,
              field: col?.field,
              sortable: col?.sortable,
              filter: col?.filter,
              editable: col?.editable,
              customButton: col?.customButton,
            };
            // Locale-aware, accent-insensitive sorting for strings. Works even when Type is Auto.
            if (col.sortable) {
              const lang = this.content?.lang || "en";
              const locale = lang === "custom" ? "en" : lang;
              const collator = new Intl.Collator(locale, {
                sensitivity: "base",
                ignorePunctuation: true,
                numeric: true,
              });
              result.comparator = (a, b) => {
                // Null/undefined/empty handling: push to bottom
                const aEmpty = a == null || a === "";
                const bEmpty = b == null || b === "";
                if (aEmpty && bEmpty) return 0;
                if (aEmpty) return 1;
                if (bEmpty) return -1;

                // Numeric handling if both look numeric
                const aNum = typeof a === "number" || (typeof a === "string" && a.trim() !== "" && !isNaN(a));
                const bNum = typeof b === "number" || (typeof b === "string" && b.trim() !== "" && !isNaN(b));
                if (aNum && bNum) return Number(a) - Number(b);

                // Locale-aware string compare (accent-insensitive)
                return collator.compare(String(a), String(b));
              };
            }
            if (col?.useCustomLabel) {
              result.valueFormatter = (params) => {
                return this.resolveMappingFormula(
                  col?.displayLabelFormula,
                  params.value
                );
              };
            }
            return result;
          }
        }
      });

      if (this.content.rowReorder && columns[0]) {
        columns[0].rowDrag = true;
      }

      return columns;
    },
    rowSelection() {
      if (this.content.rowSelection === "multiple") {
        return {
          mode: "multiRow",
          checkboxes: !this.content.disableCheckboxes,
          headerCheckbox: !this.content.disableCheckboxes,
          selectAll: this.content.selectAll || "all",
          enableClickSelection: this.content.enableClickSelection,
        };
      } else if (this.content.rowSelection === "single") {
        return {
          mode: "singleRow",
          checkboxes: !this.content.disableCheckboxes,
          enableClickSelection: this.content.enableClickSelection,
        };
      } else {
        return {
          mode: "singleRow",
          checkboxes: false,
          isRowSelectable: () => false,
          enableClickSelection: this.content.enableClickSelection,
        };
      }
    },
    style() {
      if (this.content.layout === "auto") return {};
      return {
        height: this.content.height || "400px",
      };
    },
    cssVars() {
      return {
        "--ww-data-grid_action-backgroundColor":
          this.content.actionBackgroundColor,
        "--ww-data-grid_action-color": this.content.actionColor,
        "--ww-data-grid_action-padding": this.content.actionPadding,
        "--ww-data-grid_action-border": this.content.actionBorder,
        "--ww-data-grid_action-borderRadius": this.content.actionBorderRadius,
        ...(this.content.actionFont
          ? { "--ww-data-grid_action-font": this.content.actionFont }
          : {
              "--ww-data-grid_action-fontSize": this.content.actionFontSize,
              "--ww-data-grid_action-fontFamily": this.content.actionFontFamily,
              "--ww-data-grid_action-fontWeight": this.content.actionFontWeight,
              "--ww-data-grid_action-fontStyle": this.content.actionFontStyle,
              "--ww-data-grid_action-lineHeight": this.content.actionLineHeight,
            }),
      };
    },
    theme() {
      return themeQuartz.withParams({
        headerBackgroundColor: this.content.headerBackgroundColor,
        headerTextColor: this.content.headerTextColor,
        headerFontSize: this.content.headerFontSize,
        headerFontFamily: this.content.headerFontFamily,
        headerFontWeight: this.content.headerFontWeight,
        headerHeight:
          this.content.headerHeightMode !== "auto"
            ? this.content.headerHeight
            : undefined,
        borderColor: this.content.borderColor,
        cellTextColor: this.content.cellColor,
        cellFontFamily: this.content.cellFontFamily,
        dataFontSize: this.content.cellFontSize,
        oddRowBackgroundColor: this.content.rowAlternateColor,
        backgroundColor: this.content.rowBackgroundColor,
        rowHoverColor: this.content.rowHoverColor,
        selectedRowBackgroundColor: this.content.selectedRowBackgroundColor,
        rowVerticalPaddingScale: this.content.rowVerticalPaddingScale || 1,
        menuBackgroundColor: this.content.menuBackgroundColor,
        menuTextColor: this.content.menuTextColor,
        columnHoverColor: this.content.columnHoverColor,
        foregroundColor: this.content.textColor,
        checkboxCheckedBackgroundColor: this.content.selectionCheckboxColor,
        rangeSelectionBorderColor: this.content.cellSelectionBorderColor,
        checkboxUncheckedBorderColor: this.content.checkboxUncheckedBorderColor,
        focusShadow: this.content.focusShadow?.length
          ? this.content.focusShadow
          : undefined,
        wrapperBorderRadius: this.content.wrapperBorderRadius,
      });
    },
    isEditing() {
      /* wwEditor:start */
      return (
        this.wwEditorState.editMode === wwLib.wwEditorHelper.EDIT_MODES.EDITION
      );
      /* wwEditor:end */
      // eslint-disable-next-line no-unreachable
      return false;
    },
    paginationPageSizeSelector() {
      if (
        !this.content.pagination ||
        this.content.hasPaginationSelector !== "multiple"
      ) {
        return false;
      }
      if (
        !Array.isArray(this.content.paginationPageSizeSelector) ||
        this.content.paginationPageSizeSelector.length === 0
      ) {
        return false;
      }
      return this.content.paginationPageSizeSelector;
    },
  },
  methods: {
    getRowId(params) {
      return this.resolveMappingFormula(this.content.idFormula, params.data);
    },
    onActionTrigger(event) {
      this.$emit("trigger-event", {
        name: "action",
        event,
      });
    },
    onCellValueChanged(event) {
      this.$emit("trigger-event", {
        name: "cellValueChanged",
        event: {
          oldValue: event.oldValue,
          newValue: event.newValue,
          columnId: event.column.getColId(),
          row: event.data,
        },
      });
    },
    onRowClicked(event) {
      this.$emit("trigger-event", {
        name: "rowClicked",
        event: {
          row: event.data,
          id: event.node.id,
          index: event.node.sourceRowIndex,
          displayIndex: event.rowIndex,
        },
      });
    },
    onHeaderClicked(event) {
      this.$emit("trigger-event", {
        name: "headerClicked",
        event: {
          column_id: event.column.getColId(),
          column_def: event.column.getColDef(),
          column_provided_def: event.column.getUserProvidedColDef(),
          context: event.context,
          api: event.api,
          type: event.type,
        },
      });
    },
    resetFilters() {
      if (!this.gridApi) return;
      this.gridApi.setFilterModel(null);
    },
    resetSort() {
      if (!this.gridApi) return;
      this.gridApi.applyColumnState({
        state: [],
        defaultState: { sort: null },
      });
    },
    deselectAll() {
      if (!this.gridApi) return;
      this.gridApi.deselectAll();
    },
    selectAll(mode) {
      if (!this.gridApi) return;
      if (this.content.rowSelection !== "multiple") {
        wwLib.logStore.warning(
          "Select all will have no effect, as row selection is not set to multiple"
        );
        return;
      }
      this.gridApi.selectAll(mode || this.content.selectAll || "all");
    },
    selectRow(rowId) {
      if (!this.gridApi) return;
      const rowNode = this.gridApi.getRowNode(rowId);
      if (rowNode) {
        rowNode.setSelected(true);
      }
    },
    deselectRow(rowId) {
      if (!this.gridApi) return;
      const rowNode = this.gridApi.getRowNode(rowId);
      if (rowNode) {
        rowNode.setSelected(false);
      }
    },
    /* wwEditor:start */
    generateColumns() {
      this.$emit("update:content", {
        columns: this.rowData?.[0]
          ? Object.keys(this.rowData[0]).map((key) => ({
              field: key,
              sortable: true,
              filter: true,
              customButton: false,
            }))
          : [],
      });
    },
    getOnActionTestEvent() {
      const data = this.rowData;
      if (!data || !data[0]) throw new Error("No data found");
      return {
        actionName: "actionName",
        row: data[0],
        id: 0,
        index: 0,
        displayIndex: 0,
      };
    },
    getOnCellValueChangedTestEvent() {
      const data = this.rowData;
      if (!data || !data[0]) throw new Error("No data found");
      return {
        oldValue: "oldValue",
        newValue: "newValue",
        columnId: "columnId",
        row: data[0],
      };
    },
    getSelectionTestEvent() {
      const data = this.rowData;
      if (!data || !data[0]) throw new Error("No data found");
      return {
        row: data[0],
      };
    },
    getRowClickedTestEvent() {
      const data = this.rowData;
      if (!data || !data[0]) throw new Error("No data found");
      return {
        row: data[0],
        id: 0,
        index: 0,
        displayIndex: 0,
      };
    },
    getRowDraggedTestEvent() {
      const data = this.rowData;
      if (!data || !data[0]) throw new Error("No data found");
      return {
        row: data[0],
        id: 0,
        targetIndex: 1,
        rows: data,
      };
    },
    getRowDragStartTestEvent() {
      const data = this.rowData;
      if (!data || !data[0]) throw new Error("No data found");
      return {
        row: data[0],
        id: 0,
      };
    },
    getColumnMovedTestEvent() {
      const data = this.columnDefs;
      if (!data || !data[0]) throw new Error("No data found");
      return {
        toIndex: 1,
        columnId: data[0].field,
        columnsOrder: data.map((col) => col.field),
      };
    },
    /* wwEditor:end */
  },
  /* wwEditor:start */
  watch: {
    columnDefs: {
      async handler() {
        if (this.wwEditorState?.boundProps?.columns) return;
        this.gridApi?.resetColumnState();

        if (this.wwEditorState.isACopy) return;

        // We assume there will only be one custom column each time
        const columnIndex = (this.rawContent.columns || []).findIndex(
          (col) => col && col?.cellDataType === "custom" && !col?.containerId
        );
        if (columnIndex === -1) return;
        const newColumns = [...this.rawContent.columns];
        let column = { ...newColumns[columnIndex] };
        column.containerId = await this.createElement("ww-flexbox", {
          _state: { name: `Cell ${column.headerName || column.field}` },
        });
        newColumns[columnIndex] = column;
        this.$emit("update:content:effect", { columns: newColumns });
      },
      deep: true,
    },
  },
  /* wwEditor:end */
};
</script>

<style scoped lang="scss">
.ww-datagrid {
  position: relative;
  :deep(.ag-cell-wrapper),
  :deep(.ag-cell-value) {
    height: 100%;
  }
  :deep(.ag-header-cell) {
    &.-center .ag-header-cell-label {
      justify-content: center;
    }
    &.-right {
      .ag-header-cell-label {
        justify-content: flex-end;
      }
      .ag-header-cell-filter-button {
        margin-left: 4px;
      }
    }
    &.-left .ag-header-cell-label {
      justify-content: flex-start;
    }
  }
  :deep(.ag-cell) {
    .ag-cell-value {
      display: flex;
    }

    &.-right {
      .ag-cell-value {
        justify-content: flex-end;
      }
    }
    &.-center {
      .ag-cell-value {
        justify-content: center;
      }
    }
    &.-left {
      .ag-cell-value {
        justify-content: flex-start;
      }
    }
  }
  &.loading {
    // Hide only the cells area while loading, keep headers visible
    :deep(.ag-center-cols-viewport),
    :deep(.ag-body-viewport) {
      visibility: hidden;
    }
  }
  /* wwEditor:start */
  &.editing {
    &::before {
      content: "";
      position: absolute;
      inset: 0;
      display: block;
      pointer-events: initial;
      z-index: 10;
    }
  }
  /* wwEditor:end */
}

.ww-loader-overlay {
  position: absolute;
  left: 0;
  right: 0;
  top: var(--ww-loader-top, 0px);
  bottom: 0;
  background: linear-gradient(90deg, rgba(240,240,240,0.95) 25%, rgba(250,250,250,0.95) 50%, rgba(240,240,240,0.95) 75%);
  background-size: 200% 100%;
  animation: ww-shimmer 1.2s ease-in-out infinite;
  z-index: 9;
  pointer-events: none;
}

@keyframes ww-shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

// Ensure AG Grid popups render above loader overlay
:deep(.ag-theme-quartz .ag-popup),
:deep(.ag-popup) {
  z-index: 1000 !important;
}
</style>
