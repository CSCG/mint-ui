<template>
  <div class="picker-slot {{classNames}}" :style="{ flex: flex }">
    <div v-if="!divider" v-el:wrapper class="picker-slot-wrapper" :class="{ dragging: dragging }" :style="{ height: contentHeight + 'px' }">
      <div class="picker-item" v-for="itemValue in values" :class="{ 'picker-selected': itemValue === value }">{{ itemValue }}</div>
    </div>
    <div v-if="divider">{{ content }}</div>
  </div>
</template>

<style>
  .picker-slot {
    font-size: 18px;
    overflow: hidden;
    position: relative;
    max-height: 100%
  }

  .picker-slot.picker-slot-left {
    text-align: left;
  }

  .picker-slot.picker-slot-center {
    text-align: center;
  }

  .picker-slot.picker-slot-right {
    text-align: right;
  }

  .picker-slot.picker-slot-divider {
    color: #000;
    display: -webkit-box;
    display: -ms-flexbox;
    display: -webkit-flex;
    display: flex;
    -webkit-box-align: center;
    -ms-flex-align: center;
    -webkit-align-items: center;
    align-items: center
  }

  .picker-slot-wrapper {
    -webkit-transition-duration: 0.3s;
    transition-duration: 0.3s;
    -webkit-transition-timing-function: ease-out;
    transition-timing-function: ease-out
  }

  .picker-slot-wrapper.dragging,
  .picker-slot-wrapper.dragging .picker-item {
    -webkit-transition-duration: 0s;
    transition-duration: 0s;
  }

  .picker-item {
    height: 36px;
    line-height: 36px;
    padding: 0 10px;
    white-space: nowrap;
    position: relative;
    overflow: hidden;
    text-overflow: ellipsis;
    color: #707274;
    left: 0;
    top: 0;
    width: 100%;
    box-sizing: border-box;
    -webkit-transition-duration: .3s;
    transition-duration: .3s
  }

  .picker-slot-absolute .picker-item {
    position: absolute;
  }

  .picker-item.picker-item-far {
    pointer-events: none
  }

  .picker-item.picker-selected {
    color: #000;
    -webkit-transform: translate3d(0, 0, 0) rotateX(0);
    transform: translate3d(0, 0, 0) rotateX(0);
  }

  .picker-3d .picker-items {
    overflow: hidden;
    -webkit-perspective: 700px;
    perspective: 700px;
  }

  .picker-3d .picker-item,
  .picker-3d .picker-slot,
  .picker-3d .picker-slot-wrapper {
    -webkit-transform-style: preserve-3d;
    transform-style: preserve-3d
  }

  .picker-3d .picker-slot {
    overflow: visible
  }

  .picker-3d .picker-item {
    -webkit-transform-origin: center center;
    transform-origin: center center;
    -webkit-backface-visibility: hidden;
    backface-visibility: hidden;
    -webkit-transition-timing-function: ease-out;
    transition-timing-function: ease-out
  }
</style>

<script type="text/ecmascript-6" lang="babel">
  import Vue from 'vue';
  import draggable from './draggable';
  import translateUtil from './translate';
  import { addClass, removeClass, once } from 'wind-dom';
  require('raf.js');

  var rotateElement = function(element, angle) {
    if (!element) return;
    var transformProperty = translateUtil.transformProperty;

    element.style[transformProperty] = element.style[transformProperty].replace(/rotateX\(.+?deg\)/gi, '') + ` rotateX(${angle}deg)`;
  };

  const ITEM_HEIGHT = 36;
  const VISIBLE_ITEMS_ANGLE_MAP = {
    3: -45,
    5: -20,
    7: -15
  };

  export default {
    props: {
      values: {
        type: Array,
        default() {
          return [];
        }
      },
      value: {},
      visibleItemCount: {
        type: Number,
        default: 5
      },
      rotateEffect: {
        type: Boolean,
        default: false
      },
      divider: {
        type: Boolean,
        default: false
      },
      textAlign: {
        type: String,
        default: 'center'
      },
      flex: {},
      className: {},
      content: {}
    },

    data() {
      return {
        dragging: false,
        animationFrameId: null
      };
    },

    computed: {
      classNames() {
        const PREFIX = 'picker-slot-';
        let resultArray = [];

        if (this.rotateEffect) {
          resultArray.push(PREFIX + 'absolute');
        }

        let textAlign = this.textAlign || 'center';
        resultArray.push(PREFIX + textAlign);

        if (this.divider) {
          resultArray.push(PREFIX + 'divider');
        }

        if (this.className) {
          resultArray.push(this.className);
        }

        return resultArray.join(' ');
      },
      contentHeight() {
        return ITEM_HEIGHT * this.visibleItemCount;
      },
      valueIndex() {
        return this.values.indexOf(this.value);
      },
      dragRange() {
        var values = this.values;
        var visibleItemCount = this.visibleItemCount;

        return [ -ITEM_HEIGHT * (values.length - Math.ceil(visibleItemCount / 2)), ITEM_HEIGHT * Math.floor(visibleItemCount / 2) ];
      }
    },

    methods: {
      value2Translate(value) {
        var values = this.values;
        var valueIndex = values.indexOf(value);
        var offset = Math.floor(this.visibleItemCount / 2);

        if (valueIndex !== -1) {
          return (valueIndex - offset) * -ITEM_HEIGHT;
        }
      },

      translate2Value(translate) {
        translate = Math.round(translate / ITEM_HEIGHT) * ITEM_HEIGHT;
        var index = -(translate - Math.floor(this.visibleItemCount / 2) * ITEM_HEIGHT) / ITEM_HEIGHT;

        return this.values[index];
      },

      updateRotate: function(currentTranslate, pickerItems) {
        if (this.divider) return;
        var dragRange = this.dragRange;
        var wrapper = this.$els.wrapper;

        if (!pickerItems) {
          pickerItems = wrapper.querySelectorAll('.picker-item');
        }

        if (currentTranslate === undefined) {
          currentTranslate = translateUtil.getElementTranslate(wrapper).top;
        }

        var itemsFit = Math.ceil(this.visibleItemCount / 2);
        var angleUnit = VISIBLE_ITEMS_ANGLE_MAP[this.visibleItemCount] || -20;

        [].forEach.call(pickerItems, (item, index) => {
          var itemOffsetTop = index * ITEM_HEIGHT;
          var translateOffset = dragRange[1] - currentTranslate;
          var itemOffset = itemOffsetTop - translateOffset;
          var percentage = itemOffset / ITEM_HEIGHT;

          var angle = angleUnit * percentage;
          if (angle > 180) angle = 180;
          if (angle < -180) angle = -180;

          rotateElement(item, angle);

          if (Math.abs(percentage) > itemsFit) {
            addClass(item, 'picker-item-far');
          } else {
            removeClass(item, 'picker-item-far');
          }
        });
      },

      planUpdateRotate: function() {
        var el = this.$els.wrapper;
        cancelAnimationFrame(this.animationFrameId);

        this.animationFrameId = requestAnimationFrame(() => {
          this.updateRotate();
        });

        once(el, translateUtil.transitionEndProperty, () => {
          this.animationFrameId = null;
          cancelAnimationFrame(this.animationFrameId);
        });
      },

      initEvents() {
        var el = this.$els.wrapper;
        var dragState = {};

        var velocityTranslate, prevTranslate, pickerItems;

        draggable(el, {
          start: (event) => {
            cancelAnimationFrame(this.animationFrameId);
            this.animationFrameId = null;
            dragState = {
              range: this.dragRange,
              start: new Date(),
              startLeft: event.pageX,
              startTop: event.pageY,
              startTranslateTop: translateUtil.getElementTranslate(el).top
            };
            pickerItems = el.querySelectorAll('.picker-item');
          },

          drag: (event) => {
            this.dragging = true;

            dragState.left = event.pageX;
            dragState.top = event.pageY;

            var deltaY = dragState.top - dragState.startTop;
            var translate = dragState.startTranslateTop + deltaY;

            translateUtil.translateElement(el, null, translate);

            velocityTranslate = translate - prevTranslate || translate;

            prevTranslate = translate;

            if (this.rotateEffect) {
              this.updateRotate(prevTranslate, pickerItems);
            }
          },

          end: () => {
            this.dragging = false;

            var momentumRatio = 7;
            var currentTranslate = translateUtil.getElementTranslate(el).top;
            var duration = new Date() - dragState.start;

            var momentumTranslate;
            if (duration < 300) {
              momentumTranslate = currentTranslate + velocityTranslate * momentumRatio;
            }

            var dragRange = dragState.range;

            Vue.nextTick(() => {
              var translate;
              if (momentumTranslate) {
                translate = Math.round(momentumTranslate / ITEM_HEIGHT) * ITEM_HEIGHT;
              } else {
                translate = Math.round(currentTranslate / ITEM_HEIGHT) * ITEM_HEIGHT;
              }

              translate = Math.max(Math.min(translate, dragRange[1]), dragRange[0]);

              translateUtil.translateElement(el, null, translate);

              this.value = this.translate2Value(translate);

              if (this.rotateEffect) {
                this.planUpdateRotate();
              }
            });

            dragState = {};
          }
        });
      },

      doOnValueChange() {
        var value = this.value;
        var wrapper = this.$els.wrapper;

        translateUtil.translateElement(wrapper, null, this.value2Translate(value));
      },

      doOnValuesChange() {
        var el = this.$el;
        var items = el.querySelectorAll('.picker-item');
        [].forEach.call(items, (item, index) => {
          translateUtil.translateElement(item, null, ITEM_HEIGHT * index);
        });
        if (this.rotateEffect) {
          this.planUpdateRotate();
        }
      }
    },

    ready() {
      this.ready = true;

      if (!this.divider) {
        this.initEvents();
        this.doOnValueChange();
      }

      if (this.rotateEffect) {
        this.doOnValuesChange();
      }
    },

    watch: {
      values(newVal) {
        if (this.valueIndex === -1) {
          this.value = (newVal || [])[0];
        }
        if (this.rotateEffect) {
          Vue.nextTick(() => {
            this.doOnValuesChange();
          });
        }
      },
      value() {
        this.doOnValueChange();
        if (this.rotateEffect) {
          this.planUpdateRotate();
        }
        this.$dispatch('slotValueChange', this);
      }
    }
  };
</script>
