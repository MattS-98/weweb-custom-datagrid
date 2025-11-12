<template>
  <div class="ag-cell-label-container" role="presentation">
    <div ref="eLabel" class="ag-header-cell-label" role="presentation">
      <!-- Left group: Header title + Sort icon -->
      <div class="header-left-group"
      v-on="(params.sortable) ? { click: onSortRequested } : {}">
        <span
          ref="eText"
          class="ag-header-cell-text"
        >
          {{ params?.displayName || params?.column?.colDef?.headerName || '' }}
        </span>
        <span
          v-if="params?.sortable"
          ref="eSort"
          class="ag-header-icon ag-header-label-icon ag-sort-indicator-icon"
        >
          <span
            v-if="currentSort"
            class="ag-icon"
            :class="currentSort === 'asc' ? 'ag-icon-asc' : 'ag-icon-desc'"
          ></span>
        </span>

      </div>

      <!-- Right group: Filter icon + Custom button -->
      <div class="header-right-group">
        <span
          v-if="params?.filter"
          ref="eFilter"
          @click="onFilterRequested"
          class="ag-header-icon ag-header-label-icon ag-filter-icon"
        >
          <span class="ag-icon ag-icon-filter"></span>
        </span>
        <span
          v-if="params?.customButton"
          ref="eCustomButton"
          @click="onCustomButtonClick"
          class="ag-header-icon ag-header-label-icon ag-filter-icon"
        >
          <span class="ag-icon ag-icon-filter"></span>
        </span>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'CustomHeaderComponent',
  props: {
    params: {
      type: Object,
      required: true,
    },
  },
  data() {
    return {
      currentSort: null,
    };
  },
  mounted() {
    // Get initial sort state
    this.currentSort = this.params?.column?.getSort?.() || null;

    // Listen for sort changes to update the icon
    if (this.params?.column) {
      this.params.column.addEventListener('sortChanged', () => {
        this.currentSort = this.params?.column?.getSort?.() || null;
      });
    }
  },
  methods: {
    onSortRequested(event) {
      // Trigger sorting through AG Grid's built-in sort mechanism
      if (this.params?.progressSort) {
        this.params.progressSort(event.shiftKey);
      }
    },
    onFilterRequested(event) {
      // Open the filter menu popup
      if (this.params?.showColumnMenu) {
        this.params.showColumnMenu(this.$refs.eFilter);
      }
    },
    onCustomButtonClick(event) {
      // Trigger custom button event
      if (this.params?.customButtonTrigger) {
        this.params.customButtonTrigger({
          column_id: this.params.column?.getColId?.(),
          column_def: this.params.column?.getColDef?.(),
          column_provided_def: this.params.column?.getUserProvidedColDef?.(),
          field: this.params.column?.getColDef?.()?.field,
          displayName: this.params.displayName,
        });
      }
    },
  },
};
</script>

<style>
/* Use non-scoped styles to avoid interfering with AG Grid's native hover effects */

/* Header label with space-between for left and right groups */
.ag-header-cell-label {
  display: flex !important;
  align-items: center;
  width: 100%;
  justify-content: space-between;
}

/* Left group: Header title + Sort icon - takes up remaining space */
.header-left-group {
  display: flex;
  align-items: center;
  gap: 4px;
  flex: 1;
  min-width: 0;
}

/* Right group: Filter icon + Custom button - only takes needed space */
.header-right-group {
  display: flex;
  align-items: center;
  gap: 4px;
  flex-shrink: 0;
  cursor: default;
}

/* Clickable icons */
.ag-filter-icon,
.ag-icon-menu{
  cursor: pointer;
}
</style>
