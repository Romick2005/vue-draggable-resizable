<template>
  <section class="draggable-widgets no-user-selection">
    <div v-if="widgets.length === 0">
      No widgets found
    </div>
    <div
      v-else
      ref="widgetsLayout"
      class="widgets-layout">
      <div
        v-if="activeWidgetIndex"
        :style="placeholderStyle"
        class="grid-placeholder"/>
      <div
        v-for="(widget, index) in widgets"
        v-show="isNotMaximizedMode || maximizeId === widget.id"
        :key="index"
        class="widget-box-wrapper"
        @mouseenter="onWidgetMouseEnter(index)"
        @mouseleave="onWidgetMouseLeave()">
        <vue-draggable-resizable
          :handles="handles"
          :drag-handle="dragHandle"
          :grid="grid"
          :x="maximizeId !== widget.id ? widget.x : 0"
          :y="maximizeId !== widget.id ? widget.y : 0"
          :w="maximizeId !== widget.id ? widget.width : maxWidth"
          :h="maximizeId !== widget.id ? widget.height : maxHeight"
          :style="{opacity: activeWidgetIndex === index ? 0.5 : 1}"
          :active="hoveredWidgetIndex === index"
          :draggable="isNotMaximizedMode"
          :resizable="isNotMaximizedMode"
          class="widget-box"
          @dragging="onDragging(widget, $event)"
          @dragEnd="onDragEnd(widget, $event)"
          @resizing="onResizing(widget, $event)"
          @resizeEnd="onResizeEnd(widget, $event)"
          @mouseClickDown="onMouseClickDown(widget, $event, index)"
          @mouseClickUp="onMouseClickUp()">
          <slot
            :index="index"
            :widget="widget"/>
        </vue-draggable-resizable>
      </div>
    </div>
  </section>
</template>

<script>

//import VueDraggableResizable from "vue-draggable-resizable";
import VueDraggableResizable from "vue-draggable-resizable.vue";

export default {
  components: {
    "vue-draggable-resizable": VueDraggableResizable
  },

  model: {
    prop: "widgets",
    event: "change"
  },

  props: {
    widgets: {
      type: Array,
      default: () => [],
      validator: (val) => {
        return true;//todo solid validation
      }
    },

    handles: {
      type: Array,
      default: () => ["bl", "br"],
      validator: (val) => {
        const s = new Set(["tl", "tm", "tr", "mr", "br", "bm", "bl", "ml"])
        return new Set(val.filter(h => s.has(h))).size === val.length
      }
    },

    dragHandle: {
      type: String,
      default: null
    },

    grid: {
      type: Array,
      default: () => [1, 1]
    },

    parent: {
      type: [Boolean, String],
      default: false
    },

    gridStep: {
      type: Array,
      default: () => [10, 10]
    },

    margin: {
      type: Number,
      default: 10,
      validator(val) {
        return val >= 0;
      }
    },

    maximizeId: {
      type: Number,
      default: null
    }
  },

  data() {
    return {
      activeWidgetIndex: null,
      hoveredWidgetIndex: null,
      placeholderLayout: null
    };
  },

  computed: {
    placeholderStyle() {
      const me = this;
      const placeholderLayout = me.placeholderLayout;

      return {
        position: "absolute",
        left: placeholderLayout.x + "px",
        top: placeholderLayout.y + "px",
        width: placeholderLayout.width + "px",
        height: placeholderLayout.height + "px"
      }
    },

    maxWidth() {
      const me = this;
      return me.$refs.widgetsLayout.scrollWidth;//me.$el.scrollWidth - 40;// extra space for scroll
    },

    maxHeight() {
      const me = this;
      return me.$refs.widgetsLayout.scrollHeight;
    },

    isNotMaximizedMode() {
      const me = this;
      return me.maximizeId === null;
    }
  },

  created() {
    const me = this;
  },

  methods: {
    getElementLayout($event) {
      return {
        x: $event.x,
        y: $event.y,
        width: $event.width,
        height: $event.height
      };
    },

    onResizing(widget, $event) {
      const me = this;
      me.$emit("resizing", {
        widget: widget,
        event: $event
      });
      me.updateGridPlaceholder($event, "resizing");
    },

    onDragging(widget, $event) {
      const me = this;
      me.$emit("dragging", {
        widget: widget,
        event: $event
      });
      me.updateGridPlaceholder($event, "dragging");
    },

    updateGridPlaceholder($event, changeType) {
      const me = this;
      //let {x: pX, y: pY, width: pWidth, height: pHeight} = me.placeholderLayout;
      const placeholderLayout = me.placeholderLayout;
      const pOldX = placeholderLayout.x;
      const pOldY = placeholderLayout.y;
      const pOldWidth = placeholderLayout.width;
      const pOldHeight = placeholderLayout.height;
      let pX = pOldX;
      let pY = pOldY;
      let pWidth = pOldWidth;
      let pHeight = pOldHeight;

      //let {x: eX, y: eY, width: eWidth, height: eHeight} = me.getElementLayout($event);
      const elementLayout = me.getElementLayout($event);
      let eX = elementLayout.x;
      let eY = elementLayout.y;
      let eWidth = elementLayout.width;
      let eHeight = elementLayout.height;

      const [gridStepX, gridStepY] = me.gridStep;
      const isLimitedByParent = true;
      let changeMove = {
        x: 0,
        y: 0,
        width: 0,
        height: 0
      };

      const maxWidth = me.maxWidth;
      const maxHeight = me.maxHeight;
      me.limitsLayout = {
        x: 0,
        y: 0,
        width: maxWidth,
        height: maxHeight,
      }

      const dX = eX - pX;
      if (Math.abs(dX) > gridStepX) {
        const direction = dX > 0 ? 1 : -1;
        changeMove.x = direction;
        pX = pX + Math.ceil(dX / gridStepX) * gridStepX;
        if (isLimitedByParent) {
          pX = Math.min(Math.max(0, pX), maxWidth - pOldWidth);
        }
      }

      const dY = eY - pY;
      if (Math.abs(dY) > gridStepY) {
        const direction = dY > 0 ? 1 : -1;
        changeMove.y = direction;
        pY = pY + Math.ceil(dY / gridStepY) * gridStepY;
        if (isLimitedByParent) {
          pY = Math.min(Math.max(0, pY), maxHeight - pOldHeight);
        }
      }

      const dWidth = eWidth - pWidth;
      if (Math.abs(dWidth) > gridStepX) {
        const direction = dWidth > 0 ? 1 : -1;
        changeMove.width = direction;
        pWidth = pWidth + Math.ceil(dWidth / gridStepX) * gridStepX;
        if (isLimitedByParent) {
          pWidth = Math.min(Math.max(0, pWidth), maxWidth - pOldX);
        }
      }

      const dHeight = eHeight - pHeight;
      if (Math.abs(dHeight) > gridStepY) {
        const direction = dHeight > 0 ? 1 : -1;
        changeMove.height = direction;
        pHeight = pHeight + Math.ceil(dHeight / gridStepY) * gridStepY;
        if (isLimitedByParent) {
          pHeight = Math.min(Math.max(0, pHeight), maxHeight - pOldY);
        }
      }

      if (pOldX === pX && pOldY === pY && pOldWidth === pWidth && pOldHeight === pHeight) {
        return false;
      }

      // Change placeholder layout
      placeholderLayout.x = pX;
      placeholderLayout.y = pY;
      placeholderLayout.width = pWidth;
      placeholderLayout.height = pHeight;

      // Calculate intersection
      const intersectionWidgets = me.calculateWidgetIntersection(placeholderLayout, me.widgets, me.activeWidgetIndex);
      const intersectionsCount = intersectionWidgets.length;

      // On any type of widget intersection move all intersection widget are moved down
      if (intersectionsCount > 0) {
        me.moveDown(placeholderLayout, intersectionWidgets);
      }

      // On widget move left or decrease widget width: fill free space at the right
      const placeholderMeasures = me.calculateMeasures(placeholderLayout);
      if (changeMove.x === -1 || changeMove.width === -1) {
        me.fillFreeSpaceUp(placeholderMeasures.right.borderLayout, changeType);
      }

      // On widget move right or increase widget width: fill free space at the left
      if (changeMove.x === 1 || changeMove.width === 1) {
        me.fillFreeSpaceUp(placeholderMeasures.left.borderLayout, changeType);
      }

      // On drag up move all posible widgets up or on widget height decrease
      if (changeMove.y === -1 || changeMove.height === -1) {
        me.fillFreeSpaceUp(placeholderMeasures.bottom.borderLayout ,changeType);
      }
    },

    fillFreeSpaceUp(freeSpaceLayout, changeType) {
      const me = this;
      const freeSpaceMeasures = me.calculateMeasures(freeSpaceLayout);
      const bottomWidgets = freeSpaceMeasures.bottom.widgets;
      let i = bottomWidgets.length;
      if (i === 0) {
        return false;
      }

      while (i--) {
        const widget = bottomWidgets[i];
        const widgetMeasures = me.calculateMeasures(widget);
        const y = widgetMeasures.top.borderLayout.y;
        const newTop = y + (y === 0 ? 0 : me.margin);
        if (widget.y === newTop) {
          continue;
        }
        widget.y = newTop;
        me.fireLayoutChangingEvent(widget, "up");

        const widgetBottom = widget.y + widget.height;
        const newFreeSpace = {
          x: widget.x,
          y: widgetBottom,
          width: widget.width,
          height: widgetMeasures.bottom.borderLayout.y - widgetBottom
        }
        me.fillFreeSpaceUp(newFreeSpace, changeType);
      }
    },

    moveDown(pLayout, intersectionWidgets) {
      const me = this;
      const newTop = pLayout.y + pLayout.height + me.margin;
      let i = intersectionWidgets.length;
      while (i--) {
        const widget = intersectionWidgets[i];
        widget.y = newTop;// actual widget layout change
        me.fireLayoutChangingEvent(widget, "down");
        const newIntersectionWidgets = me.calculateWidgetIntersection(widget, me.widgets, widget.id);
        if (newIntersectionWidgets.length > 0) {
          me.moveDown(widget, newIntersectionWidgets);
        }
      }
    },

    calculateWidgetIntersection(layout, widgets, skipWidgetId) {
      const left = layout.x;
      const top = layout.y;
      const right = left + layout.width;
      const bottom = top + layout.height;
      const intersectionWidgets = [];
      let i = widgets.length;
      while (i--) {
        const widget = widgets[i];

        //Check intersection and skip current widget
        const widgetX = widget.x;
        const widgetY = widget.y;
        if (
          left < widgetX + widget.width &&
          right > widgetX &&
          top < widgetY + widget.height &&
          bottom > widgetY &&
          skipWidgetId !== widget.id
        ) {
          intersectionWidgets.push(widget);
        }
      }
      return intersectionWidgets;
    },

    calculateMeasures(layout) {
      const me = this;
      const widgets = me.widgets;
      const activeWidgetIndex = me.activeWidgetIndex;
      const left = layout.x;
      const top = layout.y;
      const width = layout.width;
      const height = layout.height
      const right = left + width;
      const bottom = top + height;
      let leftWidgets = null;
      let topWidgets = null;
      let rightWidgets = null;
      let bottomWidgets = null;
      let i = widgets.length;
      while (i--) {
        let widget = widgets[i];

        //Replace widget by its placeholder
        if (activeWidgetIndex === i) {
          widget = me.placeholderLayout;
        }

        const widgetLeft = widget.x;
        const widgetTop = widget.y;
        const widgetRight = widgetLeft + widget.width;
        const widgetBottom = widgetTop + widget.height;

        // Check if widget has intersection with layout in x range
        if (left <= widgetRight && right >= widgetLeft) {

          // Find nearest vertical widgets in both directions
          if (top >= widgetBottom) { // Top
            if (topWidgets === null) {
              topWidgets = [widget];
            } else {
              const topWidget = topWidgets[0];
              const maxBottom = topWidget.y + topWidget.height;
              if (maxBottom === widgetBottom) {
                topWidgets.push(widget);
              } else if (maxBottom < widgetBottom) {
                topWidgets = [widget];
              }
            }
          } else if (bottom <= widgetTop) { // Bottom
            if (bottomWidgets === null) {
              bottomWidgets = [widget];
            } else {
              const minTop = bottomWidgets[0].y;
              if (minTop === widgetTop) {
                bottomWidgets.push(widget);
              } else if (minTop > widgetTop) {
                bottomWidgets = [widget];
              }
            }
          }
        } else if (top <= widgetBottom && bottom >= widgetTop) {// Check if widget has intersection with layout in y range

          // Find nearest horizontal widgets in both directions
          if (left >= widgetRight) { // Left
            if (leftWidgets === null) {
              leftWidgets = [widget];
            } else {
              const maxRightWidget = leftWidgets[0];
              const maxRight = maxRightWidget.x + maxRightWidget.width;
              if (maxRight === widgetRight) {
                leftWidgets.push(widget);
              } else if (maxRight < widgetRight) {
                leftWidgets = [widget];
              }
            }
          } else if (right <= widgetLeft) { // Right
            if (rightWidgets === null) {
              rightWidgets = [widget];
            } else {
              const minLeft = rightWidgets[0].x;
              if (minLeft === widgetLeft) {
                rightWidgets.push(widget);
              } else if (minLeft > widgetLeft) {
                rightWidgets = [widget];
              }
            }
          }
        }
      }

      const limitsLayout = me.limitsLayout;
      const leftX = leftWidgets === null ? limitsLayout.x : (leftWidgets[0].x + leftWidgets[0].width);
      const topY = topWidgets === null ? limitsLayout.y : (topWidgets[0].y + topWidgets[0].height);
      return {
        left: {
          widgets: leftWidgets || [],
          borderLayout: {
            x: leftX,
            y: top,
            width: left - leftX,
            height: height
          }
        },
        top: {
          widgets: topWidgets || [],
          borderLayout: {
            x: left,
            y: topY,
            width: width,
            height: top - topY
          }
        },
        right: {
          widgets: rightWidgets || [],
          borderLayout: {
            x: right,
            y: top,
            width: (rightWidgets === null ? limitsLayout.width : rightWidgets[0].x) - right,
            height: height
          }
        },
        bottom: {
          widgets: bottomWidgets || [],
          borderLayout: {
            x: left,
            y: bottom,
            width: width,
            height: (bottomWidgets === null ? limitsLayout.height : bottomWidgets[0].y) - bottom
          }
        }
      };
    },

    onResizeEnd(widget, $event) {
      const me = this;
      me.updateWidgetPosition(widget, $event, true);
    },

    onDragEnd(widget, $event) {
      const me = this;
      me.updateWidgetPosition(widget, $event, false);
    },

    updateWidgetPosition(widget, $event, isResize) {
      const me = this;
      const initialWidgetLayout = me.initialWidgetLayout;
      const placeholderLayout = me.placeholderLayout;

      const isLayoutChanged = initialWidgetLayout.x !== placeholderLayout.x ||
        initialWidgetLayout.y !== placeholderLayout.y ||
        initialWidgetLayout.width !== placeholderLayout.width ||
        initialWidgetLayout.height !== placeholderLayout.height;

      //To trigger watch options (x, y, width, height) changes on vue-draggable-resizable component (reinit on rollback)
      Object.assign(widget, me.getElementLayout($event));// Set widget current element layout

      me.$nextTick(
        function () {

          //Actual position reset/update lifecycle
          Object.assign(widget, placeholderLayout);// Set layout to the currect placeholder

          if (isLayoutChanged) {
            me.$emit("layoutChanged", {
              type: isResize ? "resizeEnd" : "dragEnd",
              widget: widget,
              widgets: me.widgets
            });
          }
          //me.placeholderLayout = null;
        }
      );
    },

    onMouseClickDown(widget, $event, index) {
      const me = this;
      me.activeWidgetIndex = index;
      me.placeholderLayout = me.getElementLayout($event);
      me.initialWidgetLayout = me.getElementLayout($event);
    },

    onMouseClickUp() {
      const me = this;
      me.activeWidgetIndex = null;
    },

    onWidgetMouseEnter(widgetIndex) {
      const me = this;
      me.hoveredWidgetIndex = widgetIndex;
    },

    onWidgetMouseLeave() {
      const me = this;

      // Fix cases when while resizing an active widget mouse pointer leave widget area
      if (me.activeWidgetIndex !== null) {
        return false;
      }
      me.hoveredWidgetIndex = null;
    },

    fireLayoutChangingEvent(widget, move) {
      const me = this;
      me.$emit("layoutChanging", {
        move: move,
        widget: widget,
        widgets: me.widgets
      });
    }
  }
};
</script>

<style lang="scss">
.draggable-widgets {
  display: flex;
  flex-grow: 1;
  flex-shrink: 0;
  flex-direction: column;
  padding: 10px 20px 20px 20px;
  background-color: #F6F6F6;

  .widgets-layout {
    display: flex;
    flex-wrap: wrap;
    overflow-y: auto;
    position: relative;
    margin-right: -20px;/*Extra space for the vertical scroll*/
    flex: 1;

    .grid-placeholder {
      background-color: #F6F6F6;
      border: 1px dashed #CCCCCC;
      border-radius: 4px;
      transition-duration: 100ms;
      z-index: 0;
      user-select: none;
    }

    .widget-box {
      display: flex;
      min-width: 100px;/*364px;*/
      min-height: 100px;/*290px;*/
      background-color: #FFFFFF;
      border-radius: 4px;
      border-width: 0;
      box-shadow: 0 1px 2px 2px #EEEEEE;
      /*padding: 5px 10px;*/
      transition-property: opacity, left, top;
      transition-duration: 0.3s;
      transition-timing-function: ease-in-out;

      &.active {
        transition: none;
      }

      .handle {
        position: absolute;
        bottom: 0;
        height: 14px;
        width: 14px;
        background-color: transparent;
        border: 0;
        z-index: 1;
        overflow: hidden;

        &:before,
        &:after {
          background-image: linear-gradient(to right, #999999 100%, #999999 100%);
          background-size: 1px 1px;
          background-position: 100%;
          background-repeat: repeat-x;
          content: "";
          display: block;
          height: 40%;
          position: absolute;
        }

        &.handle-br {
          right: 0;

          &:before,
          &:after {
            transform: rotate(135deg);
          }

          &:before {
            width: 100%;
            right: -2px;
            bottom: 2px;
          }

          &:after {
            right: 0;
            bottom: 0;
            width: 50%;
            margin-left: 25%;
          }
        }

        &.handle-bl {
          left: 0;

          &:before,
          &:after {
            transform: rotate(45deg);
          }

          &:before {
            width: 100%;
            left: -2px;
            bottom: 2px;
          }

          &:after {
            left: 0;
            bottom: 0;
            width: 50%;
            margin-right: 25%;
          }
        }
      }
    }
  }
}
</style>