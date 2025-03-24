<script>
import SortableTable from '@shell/components/SortableTable';
import ButtonDropDown from '@shell/components/ButtonDropdown';
import { isMaybeSecure } from '@shell/utils/url';
import { ingressFullPath } from '@shell/models/networking.k8s.io.ingress';

export default {
  name: 'ClusterListView',
  components: {
    SortableTable,
    ButtonDropDown,
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
    tableHeaders: {
      type: Array,
      required: true,
    },
  },
  methods: {
    isFavorited(row, favoritedApps) {
      return favoritedApps.some(
        (favoritedService) => favoritedService.id === row.id &&
          favoritedService.kind === row.kind
      );
    },
    getEndpoints(row) {
      return (
        row?.spec.ports?.map((port) => {
          const endpoint = `${isMaybeSecure(port.port, port.protocol) ? 'https' : 'http'
              }:${row.metadata.name}:${port.port}`;

          return {
            label: `${endpoint}${port.protocol === 'UDP' ? ' (UDP)' : ''}`,
            value: `/k8s/clusters/${row.clusterId}/api/v1/namespaces/` + 
              `${row.metadata.namespace}/services/${endpoint}/proxy`,
          };
        }) ?? []
      );
    },
    openLink(option) {
      window.open(option.value, '_blank');
    },
    ingressPath(row) {
      return ingressFullPath(row, row?.spec?.rules?.[0]);
    },
  },
  emits: ['toggle-favorite'],
};
</script>

<template>
  <div class="cluster-list-view">
    <div class="cluster-header">
      <h1>
        {{ clusterData.name }}
      </h1>
    </div>
    <SortableTable
      :rows="clusterData.filteredApps"
      :headers="tableHeaders"
      :row-key="'uniqueId'"
      :search="false"
      :table-actions="false"
      :row-actions="false"
      :no-rows-text="$store.getters['i18n/t']('appLauncher.noAppsFound')"
    >
      <template #cell:name="{ row }">
        {{ row.metadata.name }}
      </template>
      <template #cell:namespace="{ row }">
        {{ row.metadata.namespace }}
      </template>
      <template #cell:version="{ row }">
        {{ row.metadata.labels?.['app.kubernetes.io/version'] }}
      </template>
      <template #cell:helmChart="{ row }">
        {{ row.metadata.labels?.['helm.sh/chart'] }}
      </template>
      <template #cell:actions="{ row }">
        <div style="display: flex; justify-content: flex-end;">
          <button
            class="icon-button favorite-icon"
            @click="$emit('toggle-favorite', row)"
          >
            <i :class="['icon', isFavorited(row, favoritedApps) ? 'icon-star' : 'icon-star-open']" />
          </button>
          <a
            v-if="getEndpoints(row)?.length <= 1 && row.kind === 'Service'"
            v-bind="{disabled: getEndpoints(row)?.length > 0 ? null : true}"
            
            :href="getEndpoints(row)[0]?.value"
            :title="getEndpoints(row)?.length === 0 ? t('appLauncher.noEndpointFoundForApp')
                : t('appLauncher.launchEndpoint', {
                  endpoint: getEndpoints(row)[0].label,
                })"
            
            target="_blank"
            rel="noopener noreferrer nofollow"
            class="btn role-primary"
          >
          {{ t('appLauncher.launch') }}
          </a>
          <a
            v-else-if="row.kind === 'Ingress'"

            :href="ingressPath(row)"

            target="_blank"
            rel="noopener noreferrer nofollow"
            class="btn role-primary"
          >
          {{ t('appLauncher.launch') }}
          </a>
          <ButtonDropDown
            v-else
 
            :button-label="t('appLauncher.launch')"
            :dropdown-options="getEndpoints(row)"
            :title="t('appLauncher.launchAnEndpointFromSelection')"

            @click-action="openLink"
          />
        </div>
      </template>
    </SortableTable>
  </div>
</template>

<style scoped lang="scss">
.cluster-list-view {
  margin-bottom: 2rem;
  .favorite-icon {
    margin-right: 1rem;
    background: none;
  }
}

.cluster-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem;
  background: var(--header-bg);
  height: var(--header-height);
  position: sticky;
  top: 0;
  z-index: 1;
}
</style>
