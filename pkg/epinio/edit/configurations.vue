<script lang="ts">
import Vue, { PropType } from 'vue';
import CreateEditView from '@shell/mixins/create-edit-view';
import Loading from '@shell/components/Loading.vue';
import CruResource from '@shell/components/CruResource.vue';
import NameNsDescription from '@shell/components/form/NameNsDescription.vue';
import { mapGetters } from 'vuex';
import EpinioConfiguration from '../models/configurations';
import { EPINIO_TYPES } from '../types';
import KeyValue from '@shell/components/form/KeyValue.vue';
import { epinioExceptionToErrorsArray } from '../utils/errors';
import { validateKubernetesName } from '@shell/utils/validators/kubernetes-name';
import Banner from '@shell/components/Banner.vue';
import { sortBy } from '@shell/utils/sort';
import LabeledSelect from '@shell/components/form/LabeledSelect.vue';
import EpinioBindAppsMixin from './bind-apps-mixin.js';

interface Data {
}

export default Vue.extend<Data, any, any, any>({
  components: {
    Loading,
    CruResource,
    NameNsDescription,
    KeyValue,
    Banner,
    LabeledSelect
  },

  mixins: [CreateEditView, EpinioBindAppsMixin],

  props: {
    mode: {
      type:     String,
      required: true
    },
    value: {
      type:     Object as PropType<EpinioConfiguration>,
      required: true
    },
    initialValue: {
      type:     Object as PropType<EpinioConfiguration>,
      required: true
    }
  },

  async fetch() {
    await this.mixinFetch();

    Vue.set(this.value.meta, 'namespace', this.initialValue.meta.namespace || this.namespaces[0].metadata.name);
    this.selectedApps = [...this.initialValue.configuration?.boundapps || []];
  },

  data() {
    Vue.set(this.value, 'data', { ...this.initialValue.configuration?.details });

    return {
      errors:       [],
      selectedApps: [],
    };
  },

  computed: {
    ...mapGetters({ t: 'i18n/t' }),

    validationPassed() {
      const nameErrors = validateKubernetesName(this.value?.metadata.name || '', this.t('epinio.namespace.name'), this.$store.getters, undefined, []);

      return nameErrors.length === 0;
    },

    namespaces() {
      return sortBy(this.$store.getters['epinio/all'](EPINIO_TYPES.NAMESPACE), 'name');
    },

  },

  methods: {
    async save(saveCb: (success: boolean) => void) {
      this.errors = [];
      try {
        if (this.mode === 'create') {
          await this.value.create();
          await this.updateConfigurations();
          await this.$store.dispatch('epinio/findAll', { type: this.value.type, opt: { force: true } });
        }

        if (this.mode === 'edit') {
          await this.value.update();
          await this.updateConfigurations();
          await this.value.forceFetch();
        }

        saveCb(true);
        this.done();
      } catch (err) {
        this.errors = epinioExceptionToErrorsArray(err);
        saveCb(false);
      }
    },

    setData(data: any[]) {
      Vue.set(this.value, 'data', data);
    },

  },

  watch: {
    'value.meta.namespace'() {
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
    :min-height="'7em'"
    :mode="mode"
    :done-route="doneRoute"
    :resource="value"
    :can-yaml="false"
    :errors="errors"
    :validation-passed="validationPassed"
    @error="(e) => (errors = e)"
    @finish="save"
    @cancel="done"
  >
    <NameNsDescription
      name-key="name"
      namespace-key="namespace"
      :namespaces-override="namespaces"
      :description-hidden="true"
      :value="value.metadata"
      :mode="mode"
    />
    <div class="row">
      <div class="col span-6">
        <LabeledSelect
          v-model="selectedApps"
          :loading="$fetchState.pending"
          :disabled="noApps"
          :options="nsAppOptions"
          :searchable="true"
          :mode="mode"
          :multiple="true"
          :label-key="'epinio.configurations.bindApps.label'"
          :placeholder="noApps ? t('epinio.configurations.bindApps.placeholderNoOptions') : t('epinio.configurations.bindApps.placeholderWithOptions')"
        />
      </div>
    </div>
    <div class="spacer"></div>
    <div class="row">
      <div class="col span-11">
        <KeyValue
          :value="value.data"
          :initial-empty-row="true"
          :mode="mode"
          :title="t('epinio.configurations.pairs.label')"
          :title-protip="t('epinio.configurations.pairs.tooltip')"
          :key-label="t('epinio.applications.create.envvar.keyLabel')"
          :value-label="t('epinio.applications.create.envvar.valueLabel')"
          :parse-lines-from-file="true"
          @input="setData($event)"
        />
      </div>
    </div>
  </CruResource>
</template>
