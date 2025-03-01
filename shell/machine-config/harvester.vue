<script>
import isEmpty from 'lodash/isEmpty';
import Loading from '@shell/components/Loading';
import CreateEditView from '@shell/mixins/create-edit-view';
import LabeledSelect from '@shell/components/form/LabeledSelect';
import LabeledInput from '@shell/components/form/LabeledInput';
import UnitInput from '@shell/components/form/UnitInput';
import YamlEditor from '@shell/components/YamlEditor';
import Banner from '@shell/components/Banner';

import { get } from '@shell/utils/object';
import { mapGetters } from 'vuex';
import {
  HCI, NAMESPACE, MANAGEMENT, CONFIG_MAP, NORMAN
} from '@shell/config/types';
import { base64Decode, base64Encode } from '@shell/utils/crypto';
import { allHashSettled } from '@shell/utils/promise';
import { stringify, exceptionToErrorsArray } from '@shell/utils/error';
import { HCI as HCI_ANNOTATIONS } from '@shell/config/labels-annotations';
import { isReady } from '@shell/models/harvester/harvesterhci.io.virtualmachineimage';

export default {
  name: 'ConfigComponentHarvester',

  components: {
    Loading, LabeledSelect, LabeledInput, UnitInput, Banner, YamlEditor
  },

  mixins: [CreateEditView],

  props: {
    credentialId: {
      type:     String,
      required: true,
    },

    uuid: {
      type:     String,
      required: true,
    },

    disabled: {
      type:    Boolean,
      default: false
    },
  },

  async fetch() {
    this.errors = [];

    try {
      this.credential = await this.$store.dispatch('rancher/find', { type: NORMAN.CLOUD_CREDENTIAL, id: this.credentialId });
      const clusterId = get(this.credential, 'decodedData.clusterId');

      const url = `/k8s/clusters/${ clusterId }/v1`;

      const isImportCluster = this.credential.decodedData.clusterType === 'imported';

      this.isImportCluster = isImportCluster;

      if (clusterId && isImportCluster) {
        const res = await allHashSettled({
          namespaces:   this.$store.dispatch('cluster/request', { url: `${ url }/${ NAMESPACE }s` }),
          images:       this.$store.dispatch('cluster/request', { url: `${ url }/${ HCI.IMAGE }s` }),
          configMaps:   this.$store.dispatch('cluster/request', { url: `${ url }/${ CONFIG_MAP }s` }),
          networks:     this.$store.dispatch('cluster/request', { url: `${ url }/k8s.cni.cncf.io.network-attachment-definitions` }),
        });

        for ( const key of Object.keys(res) ) {
          const obj = res[key];

          if ( obj.status === 'rejected' ) {
            this.errors.push(stringify(obj.reason));
            continue;
          }
        }

        if (this.errors.length > 0) { // If an error is reported in the request data, see if it is due to a cluster error
          const cluster = await this.$store.dispatch('management/find', { type: MANAGEMENT.CLUSTER, id: clusterId });

          if (cluster.stateDescription && !cluster.isReady) {
            this.errors = [cluster.stateDescription];
          }
        }

        const userDataOptions = [];
        const networkDataOptions = [];

        (res.configMaps.value?.data || []).map((O) => {
          const cloudTemplate = O.metadata?.labels?.[HCI_ANNOTATIONS.CLOUD_INIT];

          if (cloudTemplate === 'user') {
            userDataOptions.push({
              label: O.metadata.name,
              value: O.data.cloudInit
            });
          }

          if (cloudTemplate === 'network') {
            networkDataOptions.push({
              label: O.metadata.name,
              value: O.data.cloudInit
            });
          }
        });

        this.userDataOptions = userDataOptions;
        this.networkDataOptions = networkDataOptions;
        this.images = res.images.value?.data;

        this.networkOptions = (res.networks.value?.data || []).map( (O) => {
          let value;
          let label;

          try {
            const config = JSON.parse(O.spec.config);

            const id = config.vlan;

            value = O.id;
            label = `${ value } (vlanId=${ id })`;
          } catch (err) {}

          return {
            label,
            value
          };
        });

        (res.namespaces.value?.data || []).forEach(async(namespace) => {
          const proxyNamespace = await this.$store.dispatch('cluster/create', namespace);

          if (!proxyNamespace.isSystem) {
            const value = namespace.metadata.name;
            const label = namespace.metadata.name;

            this.namespaceOptions.push({
              label,
              value
            });
          }
        });
      }

      if (isEmpty(this.value.cpuCount)) {
        this.value.cpuCount = '2';
      }

      if (isEmpty(this.value.memorySize)) {
        this.value.memorySize = '4';
      }

      if (isEmpty(this.value.diskSize)) {
        this.value.diskSize = '40';
      }
    } catch (e) {
      this.errors = exceptionToErrorsArray(e);
    }
  },

  data() {
    let userData = '';
    let networkData = '';

    if (this.value.userData) {
      userData = base64Decode(this.value.userData);
    }

    if (this.value.networkData) {
      networkData = base64Decode(this.value.networkData);
    }

    return {
      credential:         null,
      isImportCluster:    false,
      userData,
      networkData,
      images:             [],
      namespaceOptions:   [],
      networkOptions:     [],
      userDataOptions:    [],
      networkDataOptions: [],
      cpuCount:           ''
    };
  },

  computed: {
    ...mapGetters({ t: 'i18n/t' }),

    disabledEdit() {
      return this.disabled || !!(this.isEdit && this.value.id);
    },

    imageOptions() {
      return (this.images || []).filter( (O) => {
        return !O.spec.url.endsWith('.iso') && isReady.call(O);
      }).map( (O) => {
        const value = O.id;
        const label = `${ O.spec.displayName } (${ value })`;

        return {
          label,
          value
        };
      });
    },
  },

  watch: {
    'credentialId'() {
      if (!this.isEdit) {
        this.imageOptions = [];
        this.networkOptions = [];
        this.namespaceOptions = [];
        this.value.imageName = '';
        this.value.networkName = '';
        this.value.vmNamespace = '';
      }

      this.$fetch();
    },

    networkData(neu) {
      this.$refs.networkYamlEditor.refresh();
      this.value.networkData = base64Encode(neu);
    },

    userData(neu) {
      this.$refs.userDataYamlEditor.refresh();
      this.value.userData = base64Encode(neu);
    },
  },

  methods: {
    stringify,

    test() {
      const errors = [];

      if (!this.value.cpuCount) {
        const message = this.validatorRequiredField(this.t('cluster.credential.harvester.cpu'));

        errors.push(message);
      }

      if (!this.value.vmNamespace) {
        const message = this.validatorRequiredField(this.t('cluster.credential.harvester.namespace'));

        errors.push(message);
      }

      if (!this.value.memorySize) {
        const message = this.validatorRequiredField(this.t('cluster.credential.harvester.memory'));

        errors.push(message);
      }

      if (!this.value.diskSize) {
        const message = this.validatorRequiredField(this.t('cluster.credential.harvester.disk'));

        errors.push(message);
      }

      if (!this.value.imageName) {
        const message = this.validatorRequiredField(this.t('cluster.credential.harvester.image'));

        errors.push(message);
      }

      if (!this.value.sshUser) {
        const message = this.validatorRequiredField(this.t('cluster.credential.harvester.sshUser'));

        errors.push(message);
      }

      if (!this.value.networkName) {
        const message = this.validatorRequiredField(this.t('cluster.credential.harvester.network'));

        errors.push(message);
      }

      return { errors };
    },

    validatorRequiredField(key) {
      return this.t('validation.required', { key });
    },

    valuesChanged(value, type) {
      this.value[type] = base64Encode(value);
    },

    onOpen() {
      this.getVmImage();
    },

    async getVmImage() {
      try {
        const clusterId = get(this.credential, 'decodedData.clusterId');
        const url = `/k8s/clusters/${ clusterId }/v1`;

        if (url && this.isImportCluster) {
          const res = await this.$store.dispatch('cluster/request', { url: `${ url }/${ HCI.IMAGE }s` });

          this.images = res?.data;
        }
      } catch (e) {
        this.errors = exceptionToErrorsArray(e);
      }
    }
  },
};
</script>

<template>
  <div>
    <Loading v-if="$fetchState.pending" :delayed="true" />
    <div v-else>
      <div class="row mt-20">
        <div class="col span-6">
          <UnitInput
            v-model="value.cpuCount"
            v-int-number
            label-key="cluster.credential.harvester.cpu"
            suffix="C"
            output-as="string"
            required
            :mode="mode"
            :disabled="disabled"
            :placeholder="t('cluster.harvester.machinePool.cpu.placeholder')"
          />
        </div>

        <div class="col span-6">
          <UnitInput
            v-model="value.memorySize"
            v-int-number
            label-key="cluster.credential.harvester.memory"
            output-as="string"
            suffix="GiB"
            :mode="mode"
            :disabled="disabled"
            required
            :placeholder="t('cluster.harvester.machinePool.memory.placeholder')"
          />
        </div>
      </div>

      <div class="row mt-20">
        <div class="col span-6">
          <UnitInput
            v-model="value.diskSize"
            v-int-number
            label-key="cluster.credential.harvester.disk"
            output-as="string"
            suffix="GiB"
            :mode="mode"
            :disabled="disabled"
            required
            :placeholder="t('cluster.harvester.machinePool.disk.placeholder')"
          />
        </div>

        <div class="col span-6">
          <LabeledSelect
            v-if="isImportCluster"
            v-model="value.vmNamespace"
            :mode="mode"
            :options="namespaceOptions"
            :searchable="true"
            :required="true"
            :disabled="disabledEdit"
            label-key="cluster.credential.harvester.namespace"
            :placeholder="t('cluster.harvester.machinePool.namespace.placeholder')"
          />

          <LabeledInput
            v-else
            v-model="value.vmNamespace"
            label-key="cluster.credential.harvester.namespace"
            :required="true"
            :mode="mode"
            :disabled="disabledEdit"
            :placeholder="t('cluster.harvester.machinePool.namespace.placeholder')"
          />
        </div>
      </div>

      <div v-if="isImportCluster" class="row mt-20">
        <div class="col span-6">
          <LabeledSelect
            v-model="value.imageName"
            :mode="mode"
            :options="imageOptions"
            :required="true"
            :disabled="disabledEdit"
            label-key="cluster.credential.harvester.image"
            :placeholder="t('cluster.harvester.machinePool.image.placeholder')"
            @on-open="onOpen"
          />
        </div>

        <div class="col span-6">
          <LabeledSelect
            v-model="value.networkName"
            :mode="mode"
            :options="networkOptions"
            :required="true"
            :disabled="disabledEdit"
            label-key="cluster.credential.harvester.network"
            :placeholder="t('cluster.harvester.machinePool.network.placeholder')"
          />
        </div>
      </div>

      <div v-else class="row mt-20">
        <div class="col span-6">
          <LabeledInput
            v-model="value.imageName"
            :mode="mode"
            :required="true"
            :placeholder="t('cluster.credential.harvester.placeholder')"
            :disabled="disabledEdit"
            label-key="cluster.credential.harvester.image"
          />
        </div>

        <div class="col span-6">
          <LabeledInput
            v-model="value.networkName"
            :mode="mode"
            :required="true"
            :placeholder="t('cluster.credential.harvester.placeholder')"
            :disabled="disabledEdit"
            label-key="cluster.credential.harvester.network"
          />
        </div>
      </div>

      <div class="row mt-20">
        <div class="col span-6">
          <LabeledInput
            v-model="value.sshUser"
            label-key="cluster.credential.harvester.sshUser"
            :required="true"
            :mode="mode"
            :disabled="disabled"
            :placeholder="t('cluster.harvester.machinePool.sshUser.placeholder')"
            tooltip-key="cluster.harvester.machinePool.sshUser.toolTip"
          />
        </div>
      </div>

      <portal :to="'advanced-'+uuid">
        <h3>{{ t("cluster.credential.harvester.userData.title") }}</h3>
        <div>
          <LabeledSelect
            v-if="isImportCluster && isCreate"
            v-model="userData"
            class="mb-10"
            :options="userDataOptions"
            label-key="cluster.credential.harvester.userData.label"
            :mode="mode"
            :disabled="disabled"
          />

          <YamlEditor
            ref="userDataYamlEditor"
            :key="userData"
            class="yaml-editor mb-20"
            :editor-mode="mode === 'view' ? 'VIEW_CODE' : 'EDIT_CODE'"
            :value="userData"
            :disabled="disabled"
            @onInput="valuesChanged($event, 'userData')"
          />
        </div>

        <h3>{{ t("cluster.credential.harvester.networkData.title") }}</h3>
        <div>
          <LabeledSelect
            v-if="isImportCluster && isCreate"
            v-model="networkData"
            class="mb-10"
            :options="networkDataOptions"
            label-key="cluster.credential.harvester.networkData.label"
            :mode="mode"
            :disabled="disabled"
          />

          <YamlEditor
            ref="networkYamlEditor"
            :key="networkData"
            class="yaml-editor mb-10"
            :editor-mode="mode === 'view' ? 'VIEW_CODE' : 'EDIT_CODE'"
            :value="networkData"
            :disabled="disabled"
            @onInput="valuesChanged($event, 'networkData')"
          />
        </div>
      </portal>
    </div>
    <div v-if="errors.length">
      <div
        v-for="(err, idx) in errors"
        :key="idx"
      >
        <Banner
          color="error"
          :label="stringify(err.Message || err)"
        />
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
  $yaml-height: 200px;

  ::v-deep .yaml-editor{
    flex: 1;
    min-height: $yaml-height;
    & .code-mirror .CodeMirror {
      position: initial;
      height: auto;
      min-height: $yaml-height;
    }
  }
</style>
