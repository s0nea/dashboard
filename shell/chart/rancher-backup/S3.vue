<script>
import LabeledInput from '@shell/components/form/LabeledInput';
import Checkbox from '@shell/components/form/Checkbox';
import FileSelector from '@shell/components/form/FileSelector';
import LabeledSelect from '@shell/components/form/LabeledSelect';
import { mapGetters } from 'vuex';
export default {
  components: {
    LabeledInput,
    Checkbox,
    FileSelector,
    LabeledSelect,
  },

  props: {
    value: {
      type:    Object,
      default: () => {
        return {};
      }
    },
    mode: {
      type:    String,
      default: 'create'
    },

    secrets: {
      type:    Array,
      default: () => []
    }
  },

  computed: {
    credentialSecret: {
      get() {
        const { credentialSecretName, credentialSecretNamespace } = this.value;

        return { metadata: { name: credentialSecretName, namespace: credentialSecretNamespace } };
      },

      set(neu) {
        const { name, namespace } = neu.metadata;

        this.$set(this.value, 'credentialSecretName', name);
        this.$set(this.value, 'credentialSecretNamespace', namespace);
      }
    },
    ...mapGetters({ t: 'i18n/t' })
  },

  created() {
    const { credentialSecretName, credentialSecretNamespace } = this.value;

    if (credentialSecretName && !credentialSecretNamespace) {
      this.value.credentialSecretName = '';
    }
  },
};
</script>

<template>
  <div>
    <div class="row mb-10">
      <div class="col span-6">
        <LabeledSelect
          v-model="credentialSecret"
          :get-option-label="opt=>opt.metadata.name || ''"
          option-key="id"
          :mode="mode"
          :options="secrets"
          :label="t('backupRestoreOperator.s3.credentialSecretName')"
        />
      </div>
      <div class="col span-6">
        <LabeledInput v-model="value.bucketName" :mode="mode" :label="t('backupRestoreOperator.s3.bucketName')" />
      </div>
    </div>
    <div class="row mb-10">
      <div class="col span-6">
        <LabeledInput v-model="value.region" :mode="mode" :label="t('backupRestoreOperator.s3.region')" />
      </div>
      <div class="col span-6">
        <LabeledInput v-model="value.folder" :mode="mode" :label="t('backupRestoreOperator.s3.folder')" />
      </div>
    </div>
    <div class="row mb-10">
      <div class="col span-6">
        <LabeledInput v-model="value.endpoint" :mode="mode" :label="t('backupRestoreOperator.s3.endpoint')" />
        <Checkbox v-model="value.insecureTLSSkipVerify" class="mt-10" :mode="mode" :label="t('backupRestoreOperator.s3.insecureTLSSkipVerify')" />
      </div>
      <div class="col span-6">
        <LabeledInput v-model="value.endpointCA" :mode="mode" type="multiline" :label="t('backupRestoreOperator.s3.endpointCA')" />
        <FileSelector v-if="mode!=='view'" class="btn btn-sm role-primary mt-5" :mode="mode" :label="t('generic.readFromFile')" @selected="e=>$set(value, 'endpointCA', e)" />
      </div>
    </div>
  </div>
</template>
