<script>
import { Card } from '@components/Card';
import ButtonDropDown from '@shell/components/ButtonDropdown';
import { isMaybeSecure } from '@shell/utils/url';
import { ingressFullPath } from '@shell/models/networking.k8s.io.ingress';

export default {
  components: {
    Card,
    ButtonDropDown,
  },
  props: {
    favoritedApps: {
      type: Array,
      required: true,
    },
    app: {
      type: Object,
      default: null,
    },
    isInGlobalView: {
      type: Boolean,
      default: false,
    },
  },
  methods: {
    openLink(link) {
      window.open(link);
    },
    toggleFavorite() {
      this.$emit('toggle-favorite', this.app);
    },
  },
  computed: {
    endpoints() {
      return (
        this.app?.spec?.ports?.map((port) => {
          const endpoint = `${
            isMaybeSecure(port.port, port.protocol) ? 'https' : 'http'
          }:${this.app.metadata?.name}:${port.port}`;

          return {
            label: `${endpoint}${port.protocol === 'UDP' ? ' (UDP)' : ''}`,
            value: `/k8s/clusters/${this.app.clusterId}/api/v1/namespaces/` +
              `${this.app.metadata.namespace}/services/${endpoint}/proxy`,
          };
        }) ?? []
      );
    },
    computedServiceName() {
      return (
        this.app?.metadata?.labels?.['app.kubernetes.io/component'] ??
        this.app?.metadata?.name
      );
    },
    helmChart() {
      return this.app?.metadata?.labels?.['helm.sh/chart'];
    },
    kubernetesVersion() {
      return this.app?.metadata?.labels?.['app.kubernetes.io/version'];
    },
    name() {
      return this.app?.metadata?.name;
    },
    namespace() {
      return this.app?.metadata?.namespace;
    },
    isFavorited() {
      return this.favoritedApps.some(
        (favoritedApp) =>
          favoritedApp?.id === this.app.id &&
          favoritedApp?.kind === this.app.kind
      );
    },
    isGlobalApp() {
      return this.app?.metadata?.annotations?.['extensions.applauncher/global-app'] === 'true';
    },
    ingressPath() {
      return ingressFullPath(this.app, this.app?.spec?.rules?.[0]);
    }
  },
  name: 'AppLauncherCard',
  layout: 'plain',
};
</script>

<template>
  <Card
    v-if="app"
    class="app-launcher-card"
    :show-highlight-border="false"
    :sticky="true"
  >
    <template #title>
      <div style="width: 100%">
        <p style="font-size: 1.2rem">
          {{ app.metadata?.name }}
        </p>
        <div
          style="
            color: var(--input-label); display: flex;
            justify-content: space-between; margin-top: 4px;
          "
        >
          <p
            v-if="
              app.kind === 'Service' &&
              app.metadata?.labels?.['app.kubernetes.io/version'] !== undefined
            "
          >
            {{ kubernetesVersion }}
          </p>
          <p
            v-if="
              app.kind === 'Service' &&
              app.metadata?.labels?.['helm.sh/chart'] !== undefined
            "
          >
            {{ helmChart }}
          </p>
          <p v-if="app.kind === 'Ingress'">
            Ingress
          </p>
        </div>
      </div>
    </template>
    <template #body>
      <p v-if="app.kind === 'Service'">
        {{
          (isGlobalApp || isFavorited) &&
          isInGlobalView ? `${app.clusterName}/` : ''
        }}
        {{ namespace }}/{{ name }}
      </p>
      <p v-if="app.kind === 'Ingress'">
        {{
          (isGlobalApp || isFavorited) &&
          isInGlobalView ? `${app.clusterName}: ` : ''
        }}
        {{ ingressPath }}
      </p>
    </template>
    <template #actions>
      <button v-if="!isGlobalApp" class="icon-button" @click="toggleFavorite">
        <i :class="['icon', isFavorited ? 'icon-star' : 'icon-star-open']" />
      </button>
      <i v-else class="icon icon-globe icon-only" />
      <a
        v-if="(endpoints?.length ?? 0) <= 1 && app.kind === 'Service'"
        v-bind="{disabled: endpoints?.length > 0 ? null : true}"
        
        :href="endpoints[0]?.value"
        :title="endpoints?.length === 0 ? t('appLauncher.noEndpointFoundForApp')
                : t('appLauncher.launchEndpoint', {
                  endpoint: endpoints[0].label,
                })"
        
        target="_blank"
        rel="noopener noreferrer nofollow"
        class="btn role-primary"
      >
        {{ t('appLauncher.launch') }}
      </a>
      <a
        v-else-if="app.kind === 'Ingress'"
        :href="ingressPath"
        
        target="_blank"
        rel="noopener noreferrer nofollow"
        class="btn role-primary"
      >
        {{ t('appLauncher.launch') }}
      </a>
      <ButtonDropDown
        v-else
        
        :button-label="t('appLauncher.launch')"
        :dropdown-options="endpoints"
        :title="t('appLauncher.launchAnEndpointFromSelection')"

        @click-action="(o) => openLink(o.value)"
      />
    </template>
  </Card>
</template>

<style lang="scss" scoped>
.app-launcher-card {
  :deep() .card-body {
    overflow: hidden !important;
  }
}

.icon-button {
  background: none;
  border: none;
  cursor: pointer;
  padding: 0;
  color: var(--primary);
  font-size: 1.8rem;
  margin-right: 1rem;
}

.icon-only {
  background: none;
  border: none;
  padding: 0;
  color: var(--primary);
  font-size: 1.8rem;
  margin-right: 1rem;
  margin-top: 0.5rem;
}
</style>
