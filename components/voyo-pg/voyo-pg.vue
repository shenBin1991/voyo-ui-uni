<template>
  <view>
    <view id="paginationContent">
      <voyo-refresh
        ref="refresh"
        :enabled="isScrollUpperBound"
        @refreshStart="refresh"
      >
        <slot></slot>
        <view class="voyo-pagination-bottom-area">
          <voyo-load size="small" :show="downLoading">加载中</voyo-load>
          <view class="voyo-text-des" v-if="downNoMore">没有更多了</view>
          <view @tap="downErrorTap" class="voyo-text-des" v-if="downError"
            >加载失败，请点击重试</view
          >
        </view>
      </voyo-refresh>
    </view>
    <voyo-load
      block
      absCenterPage
      :show="initLoading"
      imgType="block"
    ></voyo-load>
    <voyo-load-error @tap="init" v-if="initError">
      {{ errorHint }}
    </voyo-load-error>
    <voyo-empty @tap="init" v-if="isEmpty">{{ emptyHint }}</voyo-empty>
  </view>
</template>
<script lang="ts">
import { isTab } from "../utils/findComponent";
import { PaginationParams } from "./voyo-pg.lib";
import { setting } from "../setting.service";

/**
 * must be child of tab component;
 */
export default {
  props: {
    upperThreshold: {
      type: Number,
      default: 0,
    },
    lowerThreshold: {
      type: Number,
      default: 30,
    },
    end: {
      type: Boolean,
      default: false,
    },
    initPullDownThreshold: {
      type: Number,
      default: 30,
    },
    errorHint: {
      type: String,
      default: "网络错误，请点击重试",
    },
    emptyHint: {
      type: String,
      default: "暂无数据",
    },
    info: undefined,
  },
  data() {
    return {
      downEnd: false,
      isEnd: false,
      isEmpty: false,
      setting: setting,
      initLoading: false,
      downLoading: false,
      refreshLoading: false,
      downNoMore: false,
      downError: false,
      initError: false,
      initEnd: false,
      isScrollUpperBound: true,
    };
  },
  watch: {
    end: {
      immediate: true,
      handler(v) {
        this.isEnd = v;
      },
    },
  },
  beforeCreate() {
    this.parentHeight = null;
    this.pgCurrentPage = 1;
    this.paginationFn = null;
    this.tab = null;
  },
  created(): void {
    /**
     * vue 在@component 下，可以传递箭头函数。
     * 而 default object下，只能传递function,function丢失this
     * 即便传递箭头，在微信小程序下依旧走不通。
     * 甚至过滤掉对象里的函数。
     * 所以，此处才用回调方式获取paginationFn.
     */
    this.$emit("getPaginationFn", (paginationFn) => {
      this.paginationFn = paginationFn;
    });
  },
  mounted(this: any) {
    if (!this.paginationFn) throw Error("voyo-pg: must call getPaginationFn");

    let tab: any = (this.tab = this.$parent);
    tab.$data.usePagination = true;
    if (!isTab(tab))
      throw new Error("voyo-pagination must be child of tab component");
    tab.$data.upperThreshold = this.upperThreshold;
    tab.$data.lowerThreshold = this.lowerThreshold;
    tab.scrollEvent.subscribe(async ({ event }) => {
      const detail = event.detail;
      this.scrollTop = detail.scrollTop;
      if (this.scrollTop === 0 && !this.isScrollUpperBound) {
        this.isScrollUpperBound = true;
      } else if (this.scrollTop !== 0 && this.isScrollUpperBound) {
        this.isScrollUpperBound = false;
      }
    });
    tab.$on("scrolltolower", () => this.pullDown());
    this.init();
  },
  methods: {
    refresh(): Promise<any> {
      const params: PaginationParams = {
        currentPage: 1,
        behavior: "refresh",
        info: this.info,
      };
      this.refreshLoad();
      return this.paginationFn(params)
        .toPromise()
        .then((result) => {
          this.handleResult(params, result);
          this.currentPage = 1;
          this.downNoMore = this.downError = false;
        })
        .catch((err) => {
          this.refreshError(err);
        })
        .finally(() => {
          this.$refs.refresh.refreshComplete();
        });
    },
    /**
     * @params
     * @res
     * @return
     * **/
    handleResult(params: PaginationParams, res: any): Boolean {
      if (setting.paginationSetting.isEmpty(params, res)) {
        this.setEmpty();
        this.refreshSuccess();
        return false;
      } else if (setting.paginationSetting.isError(params, res)) {
        if (params.behavior === "down") {
          this.downLoadFailure();
        } else if (params.behavior === "init") {
          this.initFailure();
        } else if (params.behavior === "refresh") {
          this.refreshError(res);
        }
        return false;
      } else {
        if (params.behavior === "init") {
          this.initSuccess();
        } else if (params.behavior === "refresh") {
          this.refreshSuccess();
        } else if (params.behavior === "down") {
          this.downLoadNextEnd();
          this.pgCurrentPage += 1;
        }
        if (setting.paginationSetting.isEnd(params, res)) {
          this.downLoadComplete();
        }
        return true;
      }
    },
    init(): Promise<any> {
      return this.pullInit().then((success) => {
        if (!success || this.downNoMore) return;
        // return this.initAutoPullDown();
      });
    },
    /**
     * auto pullDown after init completed;
     * if content height lower than viewHeight+initPullDownThreshold .
     */
    initAutoPullDown(): Promise<any> {
      return Promise.all([
        this.tab.getViewHeight(),
        this.getPaginationContentHeight(),
      ]).then((res: number[]) => {
        if (Math.ceil(res[0]) - res[1] > 0 - this.initPullDownThreshold) {
          return this.pullDown();
        }
      });
    },
    pullInit(): Promise<any> {
      if (this.initLoading) return Promise.resolve(false);
      this.initLoad();
      const params = {
        currentPage: this.pgCurrentPage = 1,
        behavior: "init",
        info: this.info,
      };
      this.tab.disableScrollUnOfficial();
      return this.paginationFn(params)
        .toPromise()
        .then((res) => {
          return this.handleResult(params, res);
        })
        .catch((err) => {
          console.error(err);
          this.initFailure();
          return false;
        })
        .finally(() => {
          this.tab.enableScrollUnOfficial();
        });
    },
    pullDown(): Promise<any> {
      if (this.isEnd || this.downLoading || !this.initEnd || this.downNoMore)
        return;
      this.downLoad();
      const params = {
        currentPage: this.pgCurrentPage + 1,
        behavior: "down",
        info: this.info,
      };
      return this.paginationFn(params)
        .toPromise()
        .then((res) => {
          this.handleResult(params, res);
        })
        .catch((err) => {
          this.downLoadFailure();
        });
    },
    pullRefresh() {},
    initErrorTap() {
      this.init();
    },

    reset() {
      //isEmpty
      //initError
      this.initEnd = this.downNoMore = this.downError = false;
    },
    setEmpty() {
      this.isEmpty = true;
      this.initError = false;
      this.initLoading = false;
    },
    refreshLoad() {
      this.refreshLoading = true;
    },
    refreshError(err) {
      if (!this.refreshLoading) return;
    },
    refreshSuccess() {
      if (!this.refreshLoading) return;
    },
    initLoad() {
      this.reset();
      this.initLoading = true;
    },
    initSuccess() {
      this.initLoading = false;
      this.initEnd = true;
      this.isEmpty = false;
      this.initError = false;
    },
    initFailure() {
      this.initLoading = false;
      this.initError = true;
      this.isEmpty = false;
    },
    downLoad() {
      this.downLoading = true;
    },
    downLoadNextEnd() {
      this.downLoading = false;
    },
    downLoadComplete() {
      this.downLoading = false;
      this.downNoMore = true;
    },
    downLoadFailure() {
      this.downLoading = false;
      this.downError = true;
    },
    getPaginationContentHeight() {
      return new Promise((resolve, reject) => {
        this.createSelectorQuery()
          .in(this)
          .select("#paginationContent")
          .boundingClientRect()
          .exec(([rect]) => {
            resolve(rect.height);
          });
      });
    },
    initTap() {
      this.$emit("initTap", true);
    },
    downErrorTap() {
      this.$emit("downErrorTap", true);
    },
  },
};
</script>
