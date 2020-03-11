<template>
  <transition name="cube-image-preview-fade">
    <cube-popup type="image-preview" :z-index="zIndex" :center="false" v-show="isVisible">
      <div class="cube-image-preview-container">
        <div class="cube-image-preview-header">
          <slot name="header" :current="currentPageIndex"></slot>
        </div>
        <cube-slide
          ref="slide"
          v-if="isVisible"
          :data="imgExs"
          :initial-index="currentPageIndex"
          :auto-play="false"
          :loop="loop"
          :speed="speed"
          :options="options"
          @change="slideChangeHandler"
        >
          <cube-slide-item
            v-for="(imgEx, index) in imgExs"
            :key="index"
          >
            <div class="cube-image-preview-item" @click="itemClickHandler">
              <cube-scroll
                ref="items"
                :options="scrollOptions"
                @dblclick.native="dblclickHandler(index, $event)"
              >
                <!-- 仅显示/加载 loadIndices 中索引的页面，且仅在图片加载成功后显示 -->
                <img class="cube-image-preview-img" :src="loadIndices.indexOf(index) > -1 ? (imgEx.url + '?random=' + imgEx.random) : ''" v-show="loadIndices.indexOf(index) > -1&&imgEx.isLoaded" @load="imgLoad(imgEx, index)" @error="imgError(imgEx)">
                <!-- 仅显示 loadIndices 中索引的页面加载动画 -->
                <cube-loading :size="28" v-show="loadIndices.indexOf(index) > -1&&!imgEx.isLoaded"></cube-loading>
              </cube-scroll>
            </div>
          </cube-slide-item>
          <template slot="dots"><i></i></template>
        </cube-slide>
        <div class="cube-image-preview-footer">
          <slot name="footer" :current="currentPageIndex">
            <span class="cube-image-preview-counter">{{currentPageIndex + 1}}/{{imgs.length}}</span>
          </slot>
        </div>
      </div>
    </cube-popup>
  </transition>
</template>
<script type="text/ecmascript-6">
  import CubePopup from '../popup/popup.vue'
  import CubeSlide from '../slide/slide.vue'
  import CubeSlideItem from '../slide/slide-item.vue'
  import CubeScroll from '../scroll/scroll.vue'
  import visibilityMixin from '../../common/mixins/visibility'
  import popupMixin from '../../common/mixins/popup'
  import { isAndroid } from '../../common/helpers/env'

  const COMPONENT_NAME = 'cube-image-preview'
  const EVENT_CHANGE = 'change'
  const EVENT_HIDE = 'hide'

  export default {
    name: COMPONENT_NAME,
    mixins: [visibilityMixin, popupMixin],
    props: {
      initialIndex: {
        type: Number,
        default: 0
      },
      // 图片加载失败后重新加载间隔时间，0：不重新加载
      reloadInterval: {
        type: Number,
        default: 0
      },
      imgs: {
        type: Array,
        default() {
          /* istanbul ignore next */
          return []
        }
      },
      isClickHide: {
        type: Boolean,
        default: true
      },
      loop: {
        type: Boolean,
        default: true
      },
      speed: {
        type: Number,
        default: 400
      },
      preventDefault: {
        type: Boolean,
        default: true
      }
    },
    data() {
      return {
        // 需要加载的索引
        loadIndices: [],
        // 图片对象集合，url：图片地址，isLoaded：图片加载成功标志位
        imgExs: [],
        currentPageIndex: this.initialIndex,
        options: {
          observeDOM: false,
          bounce: {
            left: true,
            right: true
          },
          useTransition: !isAndroid,
          probeType: 3,
          preventDefault: this.preventDefault
        },
        scrollOptions: {
          HWCompositing: isAndroid,
          observeDOM: false,
          zoom: true,
          bindToWrapper: true,
          freeScroll: true,
          scrollX: true,
          scrollY: true,
          probeType: 3,
          bounce: false,
          click: false,
          dblclick: true,
          bounceTime: 300,
          preventDefault: this.preventDefault
        }
      }
    },
    watch: {
      initialIndex: {
        handler: function(newIndex) {
          this.setPageIndex(newIndex)
        },
        immediate: true
      },
      imgs: {
        handler: function(val) {
          this.imgExs = []
          for (let i in val) {
            this.imgExs.push({url: val[i], isLoaded: false, random: '071123'})
          }
        },
        immediate: true
      }
    },
    methods: {
      show() {
        this.isVisible = true
        this.setPageIndex(this.initialIndex)
        this.$nextTick(() => {
          this._listenSlide()
          this._listenScroll()
        })
      },
      _listenSlide() {
        // waiting slide initial
        this.$nextTick(() => {
          const slide = this.$refs.slide.slide
          slide.on('scrollStart', this.slideScrollStartHandler)
          slide.on('scrollEnd', this.slideScrollEndHandler)
        })
      },
      _listenScroll() {
        // waiting scroll initial
        this.$nextTick(() => {
          this.$refs.items.forEach((scrollItem) => {
            const scroll = scrollItem.scroll
            scroll.on('zoomStart', this.zoomStartHandler.bind(this, scroll))
            scroll.on('beforeScrollStart', this.beforeScrollHandler)
            scroll.on('scroll', this.checkBoundary.bind(this, scroll))
            scroll.on('scrollEnd', this.scrollEndHandler.bind(this, scroll))
          })
        })
      },
      hide() {
        this.loadIndices = []
        this.isVisible = false
        this.$emit(EVENT_HIDE)
      },
      prev() {
        const slide = this.$refs.slide.slide
        slide && slide.prev()
      },
      next() {
        const slide = this.$refs.slide.slide
        slide && slide.next()
      },
      goTo(index) {
        const slide = this.$refs.slide.slide
        slide && slide.goToPage(index, 0)
      },
      imgLoad(imgEx, i) {
        /* istanbul ignore if */
        // 加载成功显示图片
        imgEx.isLoaded = true
        // 下次DOM更新时刷新缩放
        this.$nextTick(() => {
          if (this.isVisible && this.$refs.items) {
            this.$refs.items[i].scroll.refresh()
          }
        })
      },
      imgError(imgEx) {
        let timeout = this.reloadInterval
        if (timeout > 0) {
          setTimeout(() => {
            imgEx.random = new Date().getTime()
          }, timeout)
        }
      },
      setPageIndex(currentPageIndex) {
        if (this.currentPageIndex >= 0 && this.currentPageIndex !== currentPageIndex) {
          const item = this.$refs.items[this.currentPageIndex]
          if (item) {
            const scroll = item.scroll
            /* istanbul ignore if */
            if (scroll.scale !== 1) {
              scroll.scale = 1
              scroll.lastcale = 1
              scroll.refresh()
            }
          }
        }
        this.currentPageIndex = currentPageIndex
        this.addLoadIndex(currentPageIndex)
      },
      // 添加要加载的索引，当前索引及前后两个索引
      addLoadIndex(currentPageIndex) {
        let addIndices = []
        if (this.loop) {
          // 循环查看的情况
          addIndices = [
            currentPageIndex > 0 ? (currentPageIndex - 1) : (this.imgExs.length - 1),
            currentPageIndex,
            (currentPageIndex < this.imgExs.length - 1) ? (currentPageIndex + 1) : 0
          ]
        } else {
          addIndices = [currentPageIndex - 1, currentPageIndex, currentPageIndex + 1]
        }

        for (let i in addIndices) {
          if (this.loadIndices.indexOf(addIndices[i]) === -1) {
            this.loadIndices.push(addIndices[i])
          }
        }
      },
      slideChangeHandler(currentPageIndex) {
        this.setPageIndex(currentPageIndex)
        this.slideScrollEndHandler()
        this.$emit(EVENT_CHANGE, currentPageIndex)
      },
      slideScrollStartHandler() {
        const slide = this.$refs.slide.slide
        if (this._scrolling && !this._hasEnableSlide) {
          slide.disable()
        } else {
          slide.enable()
        }
      },
      slideScrollEndHandler() {
        this.$refs.items.forEach((scrollItem) => {
          this.scrollEndHandler(scrollItem.scroll)
        })
      },
      _scroll(scroll) {
        const slide = this.$refs.slide.slide
        slide.disable()
        slide.refresh()
        scroll.enable()
      },
      _slide(scroll) {
        this.$refs.slide.slide.enable()
        scroll.disable()
      },
      beforeScrollHandler() {
        // for touchstart scrollEnd
        // cancel it, do not enable slide
        clearTimeout(this.enableSlideTid)
      },
      scrollEndHandler(scroll) {
        clearTimeout(this.enableSlideTid)
        if (this.dblZooming) {
          this.dblZooming = false
          clearTimeout(this.clickTid)
        }
        this._hasEnableSlide = false
        this._scrolling = false
        scroll.enable()
        this.enableSlideTid = setTimeout(() => {
          this.$refs.slide.slide.enable()
        })
      },
      checkBoundary(scroll, pos) {
        if (scroll.distX && Math.abs(scroll.distX) > Math.abs(scroll.distY)) {
          this._scrolling = true
          const reached = scroll.distX > 0 ? pos.x >= scroll.minScrollX : pos.x <= scroll.maxScrollX
          if (reached) {
            this._hasEnableSlide = true
            this._slide(scroll)
          } else {
            if (!this._hasEnableSlide) {
              this._scroll(scroll)
            }
          }
        } else if (scroll.distY) {
          this._scrolling = true
          this._scroll(scroll)
        }
      },
      zoomStartHandler(scroll) {
        this._scroll(scroll)
      },
      dblclickHandler(index, e) {
        const scroll = this.$refs.items[index].scroll
        this.dblZooming = true
        this.zoomTo(scroll, scroll.scale > 1 ? 1 : 2, e)
        scroll.disable()
      },
      itemClickHandler() {
        clearTimeout(this.clickTid)
        this.clickTid = setTimeout(() => {
          !this.dblZooming && this.isClickHide && this.hide()
        }, this.scrollOptions.bounceTime)
      },
      zoomTo(scroll, scale, e) {
        scroll.zoomTo(scale, e.pageX, e.pageY)
      }
    },
    components: {
      CubePopup,
      CubeSlide,
      CubeSlideItem,
      CubeScroll
    }
  }
</script>
<style lang="stylus" rel="stylesheet/stylus">
  @require "../../common/stylus/variable.styl"

  .cube-image-preview-fade-enter, .cube-image-preview-fade-leave-active
    opacity: 0
  .cube-image-preview-fade-enter-active, .cube-image-preview-fade-leave-active
    transition: all .3s ease-in-out

  .cube-image-preview
    .cube-popup-mask
      opacity: .6
    .cube-popup-content
      width: 100%
      height: 100%
    .cube-slide-item
      display: flex
      align-items: center
      justify-content: center
      overflow: hidden
  .cube-image-preview-container
    height: 100%
    margin: 0 -10px
  .cube-image-preview-header,
  .cube-image-preview-footer
    position: absolute
    left: 0
    right: 0
  .cube-image-preview-header
    top: 0
  .cube-image-preview-footer
    bottom: 0
  .cube-image-preview-counter
    position: absolute
    bottom: 50px
    width: 100%
    text-align: center
    font-size: $fontsize-medium
    color: $image-preview-counter-color
  .cube-image-preview-item
    position: relative
    padding: 0 10px
    width: 100%
    height: 100%
    .cube-scroll-wrapper
      display: flex
      align-items: center
      justify-content: center
    .cube-image-preview-img
      display: block
      height: auto
      max-width: 100%
      max-height: 100%
</style>
