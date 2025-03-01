<script lang="ts">
import Vue, { PropType } from 'vue';
import ServiceInstance from '../models/service-instance';
import CreateEditView from '@shell/mixins/create-edit-view';
import CruResource from '@shell/components/CruResource.vue';
import Loading from '@shell/components/Loading.vue';
import { epinioExceptionToErrorsArray } from '../utils/errors';
import LabeledSelect from '@shell/components/form/LabeledSelect.vue';
import { EpinioCatalogServiceResource, EPINIO_TYPES } from '../types';
import { validateKubernetesName } from '@shell/utils/validators/kubernetes-name';
import { sortBy } from '@shell/utils/sort';
import NameNsDescription from '@shell/components/form/NameNsDescription.vue';
import EpinioBindAppsMixin from './bind-apps-mixin.js';
import { mapGetters } from 'vuex';

interface Data {
}

// Data, Methods, Computed, Props
export default Vue.extend<Data, any, any, any>({
  components: {
    Loading,
    CruResource,
    LabeledSelect,
    NameNsDescription
  },

  mixins: [CreateEditView, EpinioBindAppsMixin],

  props: {
    value: {
      type:     Object as PropType<ServiceInstance>,
      required: true
    },
    initialValue: {
      type:     Object as PropType<ServiceInstance>,
      required: true
    },
    mode: {
      type:     String,
      required: true
    },
  },

  async fetch() {
    await Promise.all([
      this.$store.dispatch('epinio/findAll', { type: EPINIO_TYPES.CATALOG_SERVICE }),
      this.mixinFetch()
    ]);

    Vue.set(this.value, 'namespace', this.initialValue.namespace || this.namespaces[0].metadata.name);
  },

  data() {
    return {
      errors:                 [],
      failedWaitingForDeploy: false,
    };
  },

  computed: {
    ...mapGetters({ t: 'i18n/t' }),

    validationPassed() {
      if (!this.value.catalog_service) {
        return false;
      }

      const nameErrors = validateKubernetesName(this.value?.name || '', this.t('epinio.namespace.name'), this.$store.getters, undefined, []);

      if (nameErrors.length > 0) {
        return false;
      }

      return !this.failedWaitingForDeploy;
    },

    namespaces() {
      return sortBy(this.$store.getters['epinio/all'](EPINIO_TYPES.NAMESPACE), 'name');
    },

    catalogServiceOpts() {
      return this.$store.getters['epinio/all'](EPINIO_TYPES.CATALOG_SERVICE).map((cs: EpinioCatalogServiceResource) => ({
        label: `${ cs.name } (${ cs.short_description })`,
        value: cs.name
      }));
    },

    noCatalogServices() {
      return this.catalogServiceOpts.length === 0;
    },

  },

  methods: {
    async save(saveCb: (success: boolean) => void) {
      this.errors = [];
      try {
        await this.value.create();
        if (this.selectedApps.length) {
          await this.updateServiceInstances(this.value);
        }
        await this.$store.dispatch('epinio/findAll', { type: this.value.type, opt: { force: true } });

        saveCb(true);
        this.done();
      } catch (err: Error | any) {
        if (err.message === 'waitingForDeploy') {
          Vue.set(this, 'failedWaitingForDeploy', true);
          this.errors = [this.t('epinio.serviceInstance.create.catalogService.failedWaitingForDeploy')];
        } else {
          this.errors = epinioExceptionToErrorsArray(err);
        }
        saveCb(false);
      }
    },
  },

  watch: {
    'value.namespace'() {
      Vue.set(this, 'selectedApps', []);
    }
  }

});
</script>

<template>
  <Loading v-if="!value || !namespaces" />
  <div v-else-if="!namespaces.length">
    <Banner color="warning">
      {{ t('epinio.warnings.noNamespace') }}
    </Banner>
  </div>
  <CruResource
    v-else-if="value && namespaces.length > 0"
    :can-yaml="false"
    :done-route="doneRoute"
    :mode="mode"
    :validation-passed="validationPassed"
    :resource="value"
    :errors="errors"
    @error="e=>errors = e"
    @finish="save"
  >
    <NameNsDescription
      name-key="name"
      namespace-key="namespace"
      :namespaces-override="namespaces"
      :description-hidden="true"
      :value="value"
      :mode="mode"
    />
    <div class="row">
      <div class="col span-6">
        <LabeledSelect
          v-model="value.catalog_service"
          :loading="$fetchState.pending"
          :options="catalogServiceOpts"
          :disabled="$fetchState.pending || isEdit"
          :searchable="true"
          :mode="mode"
          :multiple="false"
          :label-key="'epinio.serviceInstance.create.catalogService.label'"
          :placeholder="$fetchState.pending || noCatalogServices ? t('epinio.serviceInstance.create.catalogService.placeholderNoOptions') : t('epinio.serviceInstance.create.catalogService.placeholderWithOptions')"
          required
        />
      </div>
    </div>
    <div class="spacer"></div>
    <div class="row">
      <div class="col span-6">
        <LabeledSelect
          v-model="selectedApps"
          :loading="$fetchState.pending"
          :options="nsAppOptions"
          :disabled="noApps || $fetchState.pending"
          :searchable="true"
          :mode="mode"
          :multiple="true"
          :label-key="'epinio.configurations.bindApps.label'"
          :placeholder="$fetchState.pending || noApps ? t('epinio.configurations.bindApps.placeholderNoOptions') : t('epinio.configurations.bindApps.placeholderWithOptions')"
        />
      </div>
    </div>
  </CruResource>
</template>
