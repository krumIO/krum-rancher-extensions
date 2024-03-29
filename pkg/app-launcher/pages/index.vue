<script>
import { MANAGEMENT } from '@shell/config/types';
import Loading from '@shell/components/Loading';
import SortableTable from '@shell/components/SortableTable';
import ButtonDropDown from '@shell/components/ButtonDropdown';
import { isMaybeSecure } from '@shell/utils/url';
import { ingressFullPath } from '@shell/models/networking.k8s.io.ingress';

import AppLauncherCard from '../components/AppLauncherCard.vue';

export default {
  components: {
    Loading,
    AppLauncherCard,
    SortableTable,
    ButtonDropDown
  },
  data() {
    return {
      servicesByCluster: [],
      ingressesByCluster: [],
      clusterLoading: {},
      loading: true,
      selectedCluster: null,
      clusterOptions: [],
      selectedView: 'grid',
      favoritedServices: [],
      searchQuery: '',
      tableHeaders: [
        {
          name: 'name',
          label: 'Name',
          value: 'metadata.name',
          sort: 'metadata.name',
          sortOrder: 'asc',
        },
        {
          name: 'namespace',
          label: 'Namespace',
          value: 'metadata.namespace',
        },
        {
          name: 'version',
          label: 'Version',
          value: 'metadata.labels["app.kubernetes.io/version"]',
        },
        {
          name: 'helmChart',
          label: 'Helm Chart',
          value: 'metadata.labels["helm.sh/chart"]',
        },
        {
          name: 'actions',
          label: 'Actions',
          value: 'actions',
          align: 'right',
        },
      ],
    };
  },
  async mounted() {
    try {
      const allClusters = await this.getClusters();
      this.servicesByCluster = await this.getServicesByCluster(allClusters);
      this.ingressesByCluster = await this.getIngressesByCluster(allClusters);

      // Set the first cluster as the selected cluster
      if (this.servicesByCluster.length > 0) {
        this.selectedCluster = this.servicesByCluster[0].id;
      }
      this.generateClusterOptions();

      // Retrieve global apps based on annotations
      this.servicesByCluster.forEach((cluster) => {
        cluster.services.forEach((service) => {
          if (service.metadata?.annotations?.['extensions.applauncher/global-app'] === 'true') {
            this.favoritedServices.push({
              clusterId: cluster.id,
              service,
            });
          }
        });
      });

      // Retrieve favorites from localStorage
      const storedFavorites = localStorage.getItem('favoritedServices');
      if (storedFavorites) {
        this.favoritedServices.push(...JSON.parse(storedFavorites));
      }
    } catch (error) {
      console.error('Error fetching clusters', error);
    } finally {
      this.loading = false;
    };
  },
  methods: {
    async getClusters() {
      return await this.$store.dispatch(`management/findAll`, {
        type: MANAGEMENT.CLUSTER,
      });
    },
    getClusterName(clusterId) {
      const cluster = this.servicesByCluster.find(c => c.id === clusterId);
      return cluster ? cluster.name : '';
    },
    getCluster(clusterId) {
      return this.servicesByCluster.find(c => c.id === clusterId);
    },
    async getServicesByCluster(allClusters) {
      return await Promise.all(
        allClusters
          .filter((cluster) => cluster.isReady)
          .map(async (cluster) => {
            const clusterData = {
              name: cluster.spec.displayName,
              id: cluster.id,
              services: [],
              loading: true,
              error: false,
            };
            try {
              const services = (
                await this.$store.dispatch('cluster/request', {
                  url: `/k8s/clusters/${cluster.id}/v1/services`,
                })
              ).data;
              clusterData.services = services;
            } catch (error) {
              console.error(`Error fetching services for cluster ${cluster.id}:`, error);
              clusterData.error = true;
            } finally {
              clusterData.loading = false;
            }
            return clusterData;
          })
      );
    },
    async getIngressesByCluster(allClusters) {
      return Promise.all(
        allClusters
          .filter((cluster) => cluster.isReady)
          .map(async (cluster) => {
            const clusterData = {
              name: cluster.spec.displayName,
              id: cluster.id,
              ingresses: [],
              loading: true,
              error: false,
            };
            try {
              const ingresses = (
                await this.$store.dispatch('cluster/request', {
                  url: `/k8s/clusters/${cluster.id}/v1/networking.k8s.io.ingresses`,
                })
              ).data;
              clusterData.ingresses = ingresses;
            } catch (error) {
              console.error(`Error fetching ingresses for cluster ${cluster.id}:`, error);
              clusterData.error = true;
            } finally {
              clusterData.loading = false;
            }
            return clusterData;
          })
      );
    },
    generateClusterOptions() {
      this.clusterOptions = this.servicesByCluster.map((cluster) => ({
        label: cluster.name,
        value: cluster.id,
      }));
    },
    toggleSortOrder() {
      this.tableHeaders[0].sortOrder = this.tableHeaders[0].sortOrder === 'asc' ? 'desc' : 'asc';
    },
    getEndpoints(service) {
      return (
        service?.spec.ports?.map((port) => {
          const endpoint = `${
            isMaybeSecure(port.port, port.protocol) ? 'https' : 'http'
          }:${service.metadata.name}:${port.port}`;

          return {
            label: `${endpoint}${port.protocol === 'UDP' ? ' (UDP)' : ''}`,
            value: `/k8s/clusters/${this.selectedCluster}/api/v1/namespaces/${service.metadata.namespace}/services/${endpoint}/proxy`,
          };
        }) ?? []
      );
    },
    openLink(link) {
      window.open(link, '_blank');
    },
    toggleFavorite(service) {
      const index = this.favoritedServices.findIndex(
        (favoritedService) =>
          favoritedService.clusterId === this.selectedCluster &&
          favoritedService.service.id === service.id
      );
      if (index !== -1) {
        this.favoritedServices.splice(index, 1);
      } else {
        this.favoritedServices.push({
          clusterId: this.selectedCluster,
          service,
        });
      }
      // Store updated favorites in localStorage
      localStorage.setItem('favoritedServices', JSON.stringify(this.favoritedServices.filter((s) => (s.service.metadata.annotations?.['extensions.applauncher/global-app'] !== 'true'))));
    },
    isFavorited(service, favoritedServices) {
      return favoritedServices.some(
        (favoritedService) =>
          favoritedService.clusterId === this.selectedCluster &&
          favoritedService.service.id === service.id
      );
    },
  },
  computed: {
    selectedClusterData() {
      const cluster = this.getCluster(this.selectedCluster);
      if (cluster) {
        const ingresses = this.ingressesByCluster.find(
          (ingressCluster) => ingressCluster.id === cluster.id
        )?.ingresses || [];

        const services = cluster.services.map((service) => {
          const relatedIngress = ingresses.find((ingress) =>
            ingress.spec.rules.some((rule) =>
              rule.http.paths.some((path) =>
                path.backend.service?.name === service.metadata.name
              )
            )
          );
          return {
            ...service,
            relatedIngress,
          };
        });

        return {
          ...cluster,
          services,
          ingresses,
        };
      }
      return null;
    },
    sortedApps() {
      if (this.selectedClusterData) {
        const services = this.selectedClusterData.services.map((service) => ({
          ...service,
          type: 'service',
          uniqueId: `service-${service.id}`,
        }));

        const ingresses = this.selectedClusterData.ingresses.map((ingress) => ({
          ...ingress,
          type: 'ingress',
          uniqueId: `ingress-${ingress.id}`,
        }));

        return [...services, ...ingresses].sort((a, b) => {
          const nameA = a.metadata.name.toLowerCase();
          const nameB = b.metadata.name.toLowerCase();
          if (this.tableHeaders[0].sortOrder === 'asc') {
            return nameA.localeCompare(nameB);
          } else {
            return nameB.localeCompare(nameA);
          }
        });
      }
      return [];
    },
    filteredApps() {
      if (this.searchQuery.trim() === '') {
        return this.sortedApps;
      } else {
        const searchTerm = this.searchQuery.trim().toLowerCase();
        return this.sortedApps.filter(app => {
          return (app.metadata.name.toLowerCase().includes(searchTerm) ||
            app.metadata.namespace.toLowerCase().includes(searchTerm) ||
            app.metadata.labels?.['app.kubernetes.io/version']?.toLowerCase().includes(searchTerm) ||
            app.metadata.labels?.['helm.sh/chart']?.toLowerCase().includes(searchTerm) ||
            app.metadata.fields.includes(searchTerm));
        });
      }
    },
    sortedServices() {
      if (this.selectedClusterData) {
        return [...this.selectedClusterData.services].sort((a, b) => {
          const nameA = a.metadata.name.toLowerCase();
          const nameB = b.metadata.name.toLowerCase();
          if (this.tableHeaders[0].sortOrder === 'asc') {
            return nameA.localeCompare(nameB);
          } else {
            return nameB.localeCompare(nameA);
          }
        });
      }
      return [];
    },
  },
  layout: 'plain',
};
</script>

<template>
  <Loading v-if="loading" :label="$store.getters['i18n/t']('appLauncher.loading')" />
  <div v-else>
    <div v-if="favoritedServices.length > 0">
      <h2>{{ t('appLauncher.globalApps') }}</h2>
      <div class="services-by-cluster-grid">
        <AppLauncherCard
          v-for="favoritedService in favoritedServices"
          :key="`${favoritedService.clusterId}-${favoritedService.service.id}`"
          :cluster-id="favoritedService.clusterId"
          :service="favoritedService.service"
          :favorited-services="favoritedServices"
          @toggle-favorite="toggleFavorite"
        />
      </div>
    </div>
    <div class="cluster-header">
      <h1>{{ selectedCluster ? getClusterName(selectedCluster) : $store.getters['i18n/t']('appLauncher.selectCluster') }}</h1>
      <div class="cluster-actions">
        <button class="icon-button" @click="toggleSortOrder" v-if="selectedView === 'grid'">
          <i class="icon icon-sort" />
        </button>
        <div class="select-wrapper">
          <select v-model="selectedCluster" class="cluster-select">
            <option v-for="option in clusterOptions" :key="option.value" :value="option.value">
              {{ option.label }}
            </option>
          </select>
        </div>
        <button class="icon-button" @click="selectedView = 'grid'">
          <i class="icon icon-apps" />
        </button>
        <button class="icon-button" @click="selectedView = 'list'">
          <i class="icon icon-list-flat" />
        </button>
      </div>
    </div>
    <div v-if="selectedCluster">
      <div v-if="selectedView === 'grid'">
        <div class="search-input">
          <input v-model="searchQuery" :placeholder="$store.getters['i18n/t']('appLauncher.filter')" />
        </div>
        <div class="services-by-cluster-grid">
          <template v-if="filteredApps.length === 0">
            <p>{{ $store.getters['i18n/t']('appLauncher.noAppsFound') }}</p>
          </template>
          <AppLauncherCard
            v-else
            v-for="app in filteredApps"
            :key="app.uniqueId"
            :cluster-id="selectedCluster"
            :service="app.type === 'service' ? app : null"
            :ingress="app.type === 'ingress' ? app : null"
            :favorited-services="favoritedServices"
            @toggle-favorite="toggleFavorite"
          />
        </div>
      </div>
      <div v-else-if="selectedView === 'list'">
        <SortableTable
          :rows="sortedServices"
          :headers="tableHeaders"
          :row-key="'id'"
          :search="true"
          :table-actions="false"
          :row-actions="false"
          :no-rows-text="$store.getters['i18n/t']('appLauncher.noServicesFound')"
        >
          <template #cell:name="{row}">
            {{ row.metadata.name }}
          </template>
          <template #cell:namespace="{row}">
            {{ row.metadata.namespace }}
          </template>
          <template #cell:version="{row}">
            {{ row.metadata.labels?.['app.kubernetes.io/version'] }}
          </template>
          <template #cell:helmChart="{row}">
            {{ row.metadata.labels?.['helm.sh/chart'] }}
          </template>
          <template #cell:actions="{row}">
            <div style="display: flex; justify-content: flex-end;">
              <button class="icon-button favorite-icon" @click="toggleFavorite(row)">
                <i :class="['icon', isFavorited(row, favoritedServices) ? 'icon-star' : 'icon-star-open']" />
              </button>
              <a
                v-if="getEndpoints(row)?.length <= 1"
                :href="getEndpoints(row)[0]?.value"
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
                @click-action="(o) => openLink(o.value)"
              />
            </div>
          </template>
        </SortableTable>
      </div>
    </div>
    <div v-else>
      <p>{{ $store.getters['i18n/t']('appLauncher.noClusterSelected') }}</p>
    </div>
  </div>
</template>

<style lang="scss" scoped>
@import "@shell/assets/styles/fonts/_icons.scss";

.services-by-cluster-grid {
  display: grid;
  grid-gap: 1rem;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
}

.cluster-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 1rem;
  background: var(--header-bg);
  border-bottom: var(--header-border-size) solid var(--header-border);
  height: var(--header-height);
  position: sticky;
  top: 0;
  z-index: 1;
}

.cluster-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.icon-button {
  background: none;
  border: none;
  cursor: pointer;
  padding: 0;
  color: var(--primary);
  font-size: 1.8rem;
}

.favorite-icon {
  margin-right: 1rem;
}

.icon-button:hover {
  color: var(--primary-hover);
}

.search-input {
  margin-bottom: 1rem;
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
</style>
