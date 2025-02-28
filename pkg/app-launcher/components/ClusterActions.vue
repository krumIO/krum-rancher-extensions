<script>
export default {
    name: 'ClusterActions',
    props: {
      searchQuery: {
        type: String,
        required: true,
      },
      isGridView: {
        type: Boolean,
        required: true,
      },
      sortOrder: {
        type: String,
        required: true,
      },
      selectedCluster: {
        type: String,
        required: true,
      },
      clusterOptions: {
        type: Array,
        required: true,
      },
    },
    computed: {
      aToZorZtoA() {
        return this.sortOrder === 'asc' ? this.$store.getters['i18n/t']('appLauncher.aToZ') : this.$store.getters['i18n/t']('appLauncher.zToA');
      },
    },
    emits: ['update:search-query', 'toggle-sort', 'update:selected-cluster', 'set-view'],
};
</script>

<template>
  <div class="cluster-actions">
    <div class="search-input">
      <input
        :value="searchQuery"
        :placeholder="$store.getters['i18n/t']('appLauncher.filter')"

        @input="$emit('update:search-query', $event.target.value)"
      />
    </div>
    <div
      v-if="isGridView"
      class="sort-buttons"
      @click="$emit('toggle-sort')"
    >
      <div
        class="sort-button"
        :class="{ active: sortOrder === 'asc' }"
        :disabled="sortOrder === 'asc'"
      >
        <i class="icon-chevron-up"></i>
      </div>
      <div class="sort-label">
        <p>{{ aToZorZtoA }}</p>
      </div>
      <div
        class="sort-button"
        :class="{ active: sortOrder === 'desc' }"
        :disabled="sortOrder === 'desc'"
      >
        <i class="icon-chevron-down"></i>
      </div>
    </div>
    <div class="select-wrapper">
      <select
        :value="selectedCluster"
        class="cluster-select"
        @change="$emit('update:selected-cluster', $event.target.value)"
      >
        <option
          v-for="option in clusterOptions"
          :key="option.value"
          :value="option.value"
        >
          {{ option.label }}
        </option>
      </select>
    </div>
    <button class="icon-button" @click="$emit('set-view', 'grid')">
      <i class="icon icon-apps" />
    </button>
    <button class="icon-button" @click="$emit('set-view', 'list')">
      <i class="icon icon-list-flat" />
    </button>
  </div>
</template>

<style scoped>
.cluster-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
  position: fixed;
  right: 5.8%;
  top: 4.3rem;
  z-index: 2;
  padding-bottom: 0.425rem;
  padding-right: 4.4rem;
  background: inherit;
}

.icon-button {
  background: none;
  border: none;
  cursor: pointer;
  padding: 0;
  color: var(--primary);
  font-size: 1.8rem;
}

.sort-button {
  background: none;
  border: none;
  cursor: pointer;
  padding: none;
  color: #555555;
  font-size: 1.3rem;
  display: flex;
  align-items: center;
}

.sort-label {
  color: var(--primary);
  font-size: 1rem;
  cursor: pointer;
}

.sort-buttons {
  display: flex;
  gap: 1rem;
  align-items: center;
}

.sort-button:hover {
  color: var(--primary-hover);
}

.sort-button.active {
  color: var(--primary);
}

.search-input {
    text-align: right;
    justify-content: flex-end;
    display: flex;

    input {
        width: 190px;
        padding: 11px;
        font-size: 1rem;
        border: 1px solid var(--border);
        border-radius: 4px;
    }
}

.select-wrapper select {
    padding: 0.8rem;
    border: 1px solid #ccc;
    border-radius: 4px;
}
</style>
