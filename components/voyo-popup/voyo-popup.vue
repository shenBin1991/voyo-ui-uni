<template>
  <view>
    <view
      v-if="isShow"
      class="voyo-popup-slide"
      :class="[
        isShow && !leaving ? '__animate-' + type + '-enter' : '',
        leaving ? '__animate-' + type + '-leave' : '',
      ]"
    >
      <view @tap="tapBg" class="_layout-bg"></view>
      <view
        :class="[
          'voyo-popup-slide-view',
          '__style-' + type,
          appear ? '__appear' : '',
          appear ? '__appear-' + type : '',
          useCloseBtn ? '__width-close-icon' : '',
        ]"
      >
        <voyo-button-icon
          color="secondary"
          type="candy"
          size="mini"
          class="_close-icon"
          @tap="closeTap"
          v-if="useCloseBtn"
        >
          <text :class="[closeIconClassName]"></text>
        </voyo-button-icon>
        <slot></slot>
      </view>
    </view>
  </view>
</template>
<script>
import { setting } from "../setting.service";
export default {
  props: {
    type: {
      //type mid left top bottom right
      type: String,
      required: true,
      default: "bottom",
    },
    appear: {
      type: Boolean,
      default: true,
    },
    show: {
      type: Boolean,
      default: false,
    },
    // default close while bg be tap;
    defaultClose: {
      type: Boolean,
      default: true,
    },
    iconClose: {
      type: Boolean,
      default: true,
    },
    useCloseBtn: {
      type: Boolean,
      default: false,
    },
  },
  watch: {
    show: {
      immediate: true,
      handler(v) {
        if (v === this.isShow) return;
        v ? this.toShow() : this.toHide();
      },
    },
  },
  data() {
    return {
      isShow: false,
      leaving: false,
      closeIconClassName: setting.popupCloseIconClassName,
    };
  },
  beforeCreate() {
    this.hideTimeout = null;
  },
  methods: {
    setIsShow(v) {
      this.$emit("showChange", (this.isShow = v));
    },
    clearHideTimeout() {
      clearTimeout(this.hideTimeout);
      this.leaving = false;
      this.hideTimeout = null;
    },
    setHideTimeout() {
      this.hideTimeout = setTimeout(() => {
        this.setIsShow((this.leaving = false));
      }, 300);
    },
    toShow() {
      if (this.hideTimeout) this.clearHideTimeout();
      this.setIsShow(true);
    },
    toHide() {
      if (this.leaving || !this.isShow) return;
      this.leaving = true;
      this.setHideTimeout();
    },
    tapBg() {
      if (this.defaultClose) this.toHide();
    },
    closeTap() {
      if (this.iconClose) {
        this.toHide();
        this.$emit("tapCloseBtn", true);
      }
    },
  },
};
</script>
