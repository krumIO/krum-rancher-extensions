<script>
import AppLauncherCard from './AppLauncherCard.vue';

export default {
  name: 'ClusterGridView',
  components: {
    AppLauncherCard,
  },
  props: {
    clusterData: {
      type: Object,
      required: true,
    },
    favoritedApps: {
      type: Array,
      required: true,
    },
  },
  emits: ['toggle-favorite'],
};
</script>

<template>
  <div class="cluster-grid-view">
    <div class="cluster-header">
      <h1>
        {{ clusterData.name }}
      </h1>
    </div>
    <div class="services-by-cluster-grid">
      <AppLauncherCard
        v-for="app in clusterData.filteredApps"
        :key="`${app.clusterId}-${app.id}-${app.kind}`"
        :app="app"
        :is-in-global-view="false"
        :favorited-apps="favoritedApps"
        @toggle-favorite="$emit('toggle-favorite', $event)"
      />
    </div>
  </div>
</template>

<style scoped lang="scss">
.services-by-cluster-grid {
  display: grid;
  grid-gap: 1rem;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
}

.cluster-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin: 1rem 0;
  background: var(--header-bg);
  border-bottom: var(--header-border-size) solid var(--header-border);
  height: var(--header-height);
  position: sticky;
  top: 0;
  z-index: 1;
}

.favorite-icon {
  margin-right: 1rem;
}
</style>
