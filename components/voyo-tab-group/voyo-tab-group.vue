<template>
  <view :class="[page ? 'voyo-tab-page' : '', gentle ? 'gentle' : '']">
    <slot></slot>
  </view>
</template>
<script>
import { ExcuteAfterConnected } from "../utils/excuteAfterConnected";
export default {
  /**
   * activeIndex: number
   * tabbars
   * tabs
   * excuteAfterConnected : ExcuteAfterConnected
   */
  data() {
    return {
      componentName: "voyo-tab-group",
    };
  },
  props: {
    index: {
      type: Number,
      default: 0,
    },
    page: {
      type: Boolean,
      default: false,
    },
    gentle: {
      type: Boolean,
      default: false,
    },
  },
  watch: {
    index: {
      immediate: true,
      handler(v) {
        this.setIndex(v);
      },
    },
  },
  beforeCreate() {
    this.excuteAfterConnected = new ExcuteAfterConnected();
  },
  mounted() {
    this.$children.forEach((i) => {
      if (i.$data.componentName === "voyo-tabs") this.tabs = i;
      if (i.$data.componentName === "voyo-tabbars") this.tabbars = i;
    });
    if (!this.tabs || !this.tabbars)
      throw new Error("A lack of tabs or tabbars");
    this.tabbars.$on("input", (index) => {
      this.setIndex(index);
    });

    this.excuteAfterConnected.connect();
  },
  methods: {
    setIndex(v) {
      this.excuteAfterConnected.execute(() => {
        if (v === this.activeIndex) return;
        this.activeIndex = v;
        this.tabs && this.tabs.selectActiveTab(v);
        this.tabbars && this.tabbars.setIndex(v);
        this.$emit("input", v);
      });
    },
    listenTabbarsChange(e) {},
    mountedPromise() {
      return new Promise((resolve, reject) => {
        this.excuteAfterConnected.execute(() => resolve());
      });
    },
  },
};
</script>
<style lang="scss"></style>
