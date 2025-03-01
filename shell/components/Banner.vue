<script lang="ts">
import Vue from 'vue';
import { nlToBr } from '@shell/utils/string';
import { stringify } from '@shell/utils/error';

export default Vue.extend({
  props: {
    color: {
      type:    String,
      default: 'secondary'
    },
    label: {
      type:    [String, Error, Object],
      default: null,
    },
    labelKey: {
      type:    String,
      default: null,
    },
    closable: {
      type:    Boolean,
      default: false,
    },
    stacked: {
      type:    Boolean,
      default: false,
    }
  },
  computed: {
    /**
     * Return message text as label
     */
    messageLabel(): string | void {
      return !(typeof this.label === 'string') ? stringify(this.label) : undefined;
    }
  },
  methods: { nlToBr },
});
</script>
<template>
  <div
    class="banner"
    :class="{
      [color]: true,
      closable,
      stacked
    }"
  >
    <slot>
      <t v-if="labelKey" :k="labelKey" :raw="true" />
      <span v-else-if="messageLabel">{{ messageLabel }}</span>
      <span v-else v-html="nlToBr(label)" />
    </slot>
    <div v-if="closable" class="closer" @click="$emit('close')">
      <i class="icon icon-2x icon-close closer-icon" />
    </div>
  </div>
</template>

<style lang="scss" scoped>
  $left-border-size: 4px;

  .banner {
    padding: 10px;
    margin: 15px 0;
    width: 100%;
    transition: all 0.2s ease;
    position: relative;
    line-height: 20px;

    &.stacked {
      padding: 0 10px;
      margin: 0;
      transition: none;

      &:first-child {
        padding-top: 10px;
      }
      &:last-child {
        padding-bottom: 10px;
      }
    }

    &.closable {
      padding-right: 40px;
    }

    .closer {
      display: flex;
      align-items: center;

      cursor: pointer;
      position: absolute;
      top: 0;
      right: 0;
      bottom: 0;
      width: 40px;
      line-height: 42px;
      text-align: center;

      .closer-icon {
        font-size: 22px;
        opacity: 0.7;

        &:hover {
          opacity: 1;
          color: var(--link);
        }
      }
    }

    &.primary {
      background: var(--primary);
      border-left: solid $left-border-size var(--primary);
      color: var(--body-text);
    }

    &.secondary {
      background: var(--secondary-banner-bg);
      border-left: solid $left-border-size var(--secondary);
      color: var(--body-text);
    }

    &.success {
      background: var(--success-banner-bg);
      border-left: solid $left-border-size var(--success);
      color: var(--body-text);
    }

    &.info {
      background: var(--info-banner-bg);
      border-left: solid $left-border-size var(--info);
      color: var(--body-text);
    }

    &.warning {
      background: var(--warning-banner-bg);
      border-left: solid $left-border-size var(--warning);
      color: var(--body-text);
    }

    &.error {
      background: var(--error-banner-bg);
      border-left: solid $left-border-size var(--error);
      color: var(--error);
    }
  }
</style>
