<template>
  <div
    :style="style"
    :class="{
      active: enabled,
      draggable: draggable,
      resizable: resizable,
      dragging: dragging,
      resizing: resizing}"
    class="vdr"
    @mousedown.stop="elementDown"
    @touchstart.prevent.stop="elementTouchDown">
    <div
      v-for="handle in actualHandles"
      :key="handle"
      :class="'handle-' + handle"
      :style="{display: enabled ? 'block' : 'none'}"
      class="handle"
      @mousedown.stop.prevent="handleDown(handle, $event)"
      @touchstart.stop.prevent="handleTouchDown(handle, $event)">
      <slot :name="handle"/>
    </div>
    <slot/>
  </div>
</template>

<script>
import {
  matchesSelectorToParentElements,
  addEvent,
  removeEvent
} from "../utils";

const events = {
  mouse: {
    start: "mousedown",
    move: "mousemove",
    stop: "mouseup"
  },
  touch: {
    start: "touchstart",
    move: "touchmove",
    stop: "touchend"
  }
};

const userSelectNone = {
  userSelect: "none",
  MozUserSelect: "none",
  WebkitUserSelect: "none",
  MsUserSelect: "none"
};

const userSelectAuto = {
  userSelect: "auto",
  MozUserSelect: "auto",
  WebkitUserSelect: "auto",
  MsUserSelect: "auto"
};

let eventsFor = events.mouse;

export default {
  replace: true,
  name: "vue-draggable-resizable",
  props: {
    debug: {
      type: Boolean,
      default: false
    },

    disableUserSelect: {
      type: Boolean,
      default: true
    },

    enableNativeDrag: {
      type: Boolean,
      default: false
    },

    preventDeactivation: {
      type: Boolean,
      default: false
    },

    active: {
      type: Boolean,
      default: false
    },

    draggable: {
      type: Boolean,
      default: true
    },

    resizable: {
      type: Boolean,
      default: true
    },

    lockAspectRatio: {
      type: Boolean,
      default: false
    },

    w: {
      type: Number,
      default: 200,
      validator: (val) => val >= 0
    },

    h: {
      type: Number,
      default: 200,
      validator: (val) => val >= 0
    },

    minWidth: {
      type: Number,
      default: 0,
      validator: (val) => val >= 0
    },

    minHeight: {
      type: Number,
      default: 0,
      validator: (val) => val >= 0
    },

    x: {
      type: Number,
      default: 0,
      validator: (val) => typeof val === "number"
    },

    y: {
      type: Number,
      default: 0,
      validator: (val) => typeof val === "number"
    },

    z: {
      type: [String, Number],
      default: "auto",
      validator: (val) => (typeof val === "string" ? val === "auto" : val >= 0)
    },

    handles: {
      type: Array,
      default: () => ["tl", "tm", "tr", "mr", "br", "bm", "bl", "ml"],
      validator: (val) => {
        const s = new Set(["tl", "tm", "tr", "mr", "br", "bm", "bl", "ml"])
        return new Set(val.filter(h => s.has(h))).size === val.length
      }
    },

    dragHandle: {
      type: String,
      default: null
    },

    dragCancel: {
      type: String,
      default: null
    },

    axis: {
      type: String,
      default: "both",
      validator: (val) => ["x", "y", "both"].includes(val)
    },

    grid: {
      type: Array,
      default: () => [1, 1]
    },

    parent: {
      type: [Boolean, String],
      default: false
    }
  },

  data() {
    const me = this;
    const y = me.y;
    const x = me.x;
    const w = me.w;
    const h = me.h;
    return {
      rawWidth: w,
      rawHeight: h,
      rawLeft: x,
      rawTop: y,
      rawRight: null,
      rawBottom: null,

      left: x,
      top: y,
      right: null,
      bottom: null,

      aspectFactor: w / h,

      parentWidth: null,
      parentHeight: null,

      minW: me.minWidth,
      minH: me.minHeight,

      handle: null,
      enabled: me.active,
      resizing: false,
      dragging: false,
      zIndex: me.z
    }
  },

  created() {
    const me = this;
    me.resetBoundsAndMouseState();
  },

  mounted() {
    const me = this;
    if (!me.enableNativeDrag) {
      me.$el.ondragstart = () => false;
    }

    [me.parentWidth, me.parentHeight] = me.getParentSize();

    me.rawRight = me.parentWidth - me.rawWidth - me.rawLeft;
    me.rawBottom = me.parentHeight - me.rawHeight - me.rawTop;

    const documentElement = document.documentElement;
    addEvent(documentElement, "mousedown", me.deselect);//events.mouse.start
    addEvent(documentElement, "touchend touchcancel", me.deselect);//events.touch.stop
    addEvent(window, "resize", me.checkParentSize);
  },

  beforeDestroy() {
    const me = this;
    const documentElement = document.documentElement;
    removeEvent(documentElement, "mouseup", me.handleUp);//events.mouse.stop
    removeEvent(documentElement, "touchstart", me.handleUp);//events.touch.start
    removeEvent(documentElement, "mousemove", me.move);//events.mouse.move
    removeEvent(documentElement, "touchmove", me.move);//events.touch.move

    removeEvent(documentElement, "mousedown", me.deselect);//events.mouse.start
    removeEvent(documentElement, "touchend touchcancel", me.deselect);//events.touch.stop
    removeEvent(window, "resize", me.checkParentSize);
  },

  methods: {
    elementTouchDown(e) {
      const me = this;
      eventsFor = events.touch;
      me.elementDown(e);
    },

    elementDown(e) {
      const me = this;
      const target = e.target || e.srcElement;
      const el = me.$el;
      if (el.contains(target)) {
        if (
          (me.dragHandle && !matchesSelectorToParentElements(target, me.dragHandle, el)) ||
          (me.dragCancel && matchesSelectorToParentElements(target, me.dragCancel, el))
        ) {
          return false;
        }

        if (!me.enabled) {
          me.enabled = true;

          me.$emit("activated");
          me.$emit("update:active", true);
        }

        if (me.draggable) {
          me.dragging = true;
        }

        me.prepareStartingPoint(e);
      }
    },

    handleTouchDown(handle, e) {
      const me = this;
      eventsFor = events.touch;
      me.handleDown(handle, e);
    },

    handleDown(handle, e) {
      const me = this;
      if (e.stopPropagation) {
        e.stopPropagation();
      }

      // Here we avoid a dangerous recursion by faking
      // corner handles as middle handles
      if (me.lockAspectRatio && !handle.includes("m")) {
        me.handle = "m" + handle.substring(1);
      } else {
        me.handle = handle;
      }

      me.resizing = true;

      me.prepareStartingPoint(e);
    },

    move(e) {
      const me = this;
      if (me.resizing) {
        me.handleMove(e);
      } else if (me.dragging) {
        me.elementMove(e);
      }
    },

    elementMove(e) {
      const me = this;
      const axis = me.axis;
      const grid = me.grid;
      const mouseClickPosition = me.mouseClickPosition;

      const tmpDeltaX = axis && axis !== "y" ? mouseClickPosition.mouseX - (e.touches ? e.touches[0].pageX : e.pageX) : 0
      const tmpDeltaY = axis && axis !== "x" ? mouseClickPosition.mouseY - (e.touches ? e.touches[0].pageY : e.pageY) : 0

      const [deltaX, deltaY] = me.snapToGrid(me.grid, tmpDeltaX, tmpDeltaY)

      if (!deltaX && !deltaY) {
        return false;
      }

      me.rawTop = mouseClickPosition.top - deltaY;
      me.rawBottom = mouseClickPosition.bottom + deltaY;
      me.rawLeft = mouseClickPosition.left - deltaX;
      me.rawRight = mouseClickPosition.right + deltaX;

      if (me.firstMove) {
        me.fireEvent("dragStart");
        me.firstMove = false;
      }
      me.fireEvent("dragging");
    },

    handleMove(e) {
      const me = this;
      const handle = me.handle;
      const mouseClickPosition = me.mouseClickPosition;

      const tmpDeltaX = mouseClickPosition.mouseX - (e.touches ? e.touches[0].pageX : e.pageX);
      const tmpDeltaY = mouseClickPosition.mouseY - (e.touches ? e.touches[0].pageY : e.pageY);

      const [deltaX, deltaY] = me.snapToGrid(me.grid, tmpDeltaX, tmpDeltaY);

      if (!deltaX && !deltaY) {
        return false;
      }

      if (handle.includes("b")) {
        me.rawBottom = mouseClickPosition.bottom + deltaY;
      } else if (handle.includes("t")) {
        me.rawTop = mouseClickPosition.top - deltaY;
      }

      if (handle.includes("r")) {
        me.rawRight = mouseClickPosition.right + deltaX;
      } else if (handle.includes("l")) {
        me.rawLeft = mouseClickPosition.left - deltaX;
      }

      if (me.firstMove) {
        me.fireEvent("resizeStart");
        me.firstMove = false;
      }
      me.fireEvent("resizing");
    },

    handleUp(e) {
      const me = this;
      me.handle = null;

      me.rawTop = me.top;
      me.rawBottom = me.bottom;
      me.rawLeft = me.left;
      me.rawRight = me.right;

      if (me.resizing) {
        me.resizing = false;
        if (me.firstMove === false) {
          me.fireEvent("resizeEnd");
        }
      }
      if (me.dragging) {
        me.dragging = false;
        if (me.firstMove === false) {
          me.fireEvent("dragEnd");
        }
      }
      me.resetBoundsAndMouseState();
      me.fireEvent("mouseClickUp");
    },

    resetBoundsAndMouseState() {
      const me = this;

      // No need to reinitialize this as it is populated only on click and used only during move
      //me.mouseClickPosition = null;
      /*me.mouseClickPosition = {
        mouseX: 0,
        mouseY: 0,
        left: 0,
        top: 0,
        right: 0,
        bottom: 0
      };*/

      // Also populated on first click and cam ne used in calculated properties
      //me.bounds = null;
      me.bounds = {
        minLeft: null,
        maxLeft: null,
        minRight: null,
        maxRight: null,
        minTop: null,
        maxTop: null,
        minBottom: null,
        maxBottom: null
      };

      //me.firstMove = null;//also not necessary
      //Possible values
      //null - initial,
      //true - before any position changes, but with attached move listeners,
      //false - after mouse position changes

      const documentElement = document.documentElement;
      removeEvent(documentElement, eventsFor.move, me.handleMove);
      removeEvent(documentElement, eventsFor.stop, me.handleUp);
    },

    checkParentSize() {
      const me = this;
      if (me.parent) {
        const [newParentWidth, newParentHeight] = me.getParentSize();

        const deltaX = me.parentWidth - newParentWidth;
        const deltaY = me.parentHeight - newParentHeight;

        me.rawRight -= deltaX;
        me.rawBottom -= deltaY;

        me.parentWidth = newParentWidth;
        me.parentHeight = newParentHeight;
      }
    },

    getParentSize() {
      const me = this;
      const parent = me.parent;

      if (parent === true) {
        const parentNode = me.$el.parentNode;
        return [
          parentNode.offsetWidth,
          parentNode.offsetHeight
        ];
      }

      if (typeof parent === "string") {
        const parentNode = document.querySelector(parent);//TODO some error on SSR
        if (!(parentNode instanceof HTMLElement)) {
          throw new Error(`The selector ${parent} does not match any element`)
        }
        return [parentNode.offsetWidth, parentNode.offsetHeight];
      }
      return [null, null];
    },

    deselect(e) {
      const me = this;
      const target = e.target || e.srcElement;
      const regex = new RegExp("vdr" + "-([trmbl]{2})", "");

      if (!me.$el.contains(target) && !regex.test(target.className)) {
        if (me.enabled && !me.preventDeactivation) {
          me.enabled = false;
          me.$emit("deactivated");
          me.$emit("update:active", false);
        }
      }

      me.resetBoundsAndMouseState();
    },

    prepareStartingPoint(e) {
      const me = this;
      const touches = e.touches;
      me.mouseClickPosition = {
        mouseX: touches ? touches[0].pageX : e.pageX,
        mouseY: touches ? touches[0].pageY : e.pageY,
        left: me.left,
        right: me.right,
        top: me.top,
        bottom: me.bottom
      }

      if (me.resizing) {
        me.bounds = me.calcResizeLimits();
      } else if (me.dragging && me.parent) {
        me.bounds = me.calcDragLimits();
      }

      me.firstMove = true;
      const documentElement = document.documentElement;
      addEvent(documentElement, eventsFor.move, me.move);
      addEvent(documentElement, eventsFor.stop, me.handleUp);
      me.fireEvent("mouseClickDown");
    },

    calcResizeLimits() {
      const me = this;
      let minW = me.minW;
      let minH = me.minH;

      const aspectFactor = me.aspectFactor;
      const [gridX, gridY] = me.grid;
      const width = me.width;
      const height = me.height;
      const bottom = me.bottom;
      const top = me.top;
      const left = me.left;
      const right = me.right;

      if (me.lockAspectRatio) {
        if (minW / minH > aspectFactor) {
          minH = minW / aspectFactor;
        } else {
          minW = aspectFactor * minH;
        }
      }

      const limits = {
        minLeft: me.parent ? (me.parentWidth + left) % gridX : null,
        maxLeft: left + Math.floor((width - minW) / gridX) * gridX,
        minRight: me.parent ? (me.parentWidth + right) % gridX : null,
        maxRight: right + Math.floor((width - minW) / gridX) * gridX,
        minTop: me.parent ? (me.parentHeight + top) % gridY : null,
        maxTop: top + Math.floor((height - minH) / gridY) * gridY,
        minBottom: me.parent ? (me.parentHeight + bottom) % gridY : null,
        maxBottom: bottom + Math.floor((height - minH) / gridY) * gridY
      };

      if (me.lockAspectRatio && me.parent) {
        const aspectRatioLimits = {
          minLeft: left - top * aspectFactor,
          minRight: right - bottom * aspectFactor,
          minTop: top - left / aspectFactor,
          minBottom: bottom - right / aspectFactor
        };

        limits.minTop = Math.max(limits.minTop, aspectRatioLimits.minTop);
        limits.minLeft = Math.max(limits.minLeft, aspectRatioLimits.minLeft);
        limits.minRight = Math.max(limits.minRight, aspectRatioLimits.minRight);
        limits.minBottom = Math.max(limits.minBottom, aspectRatioLimits.minBottom);
      }

      return limits;
    },

    calcDragLimits() {
      const me = this;
      const [gridX, gridY] = me.grid;
      return {
        minLeft: (me.parentWidth + me.left) % gridX,
        maxLeft: Math.floor((me.parentWidth - me.width - me.left) / gridX) * gridX + me.left,
        minRight: (me.parentWidth + me.right) % gridX,
        maxRight: Math.floor((me.parentWidth - me.width - me.right) / gridX) * gridX + me.right,
        minTop: (me.parentHeight + me.top) % gridY,
        maxTop: Math.floor((me.parentHeight - me.height - me.top) / gridY) * gridY + me.top,
        minBottom: (me.parentHeight + me.bottom) % gridY,
        maxBottom: Math.floor((me.parentHeight - me.height - me.bottom) / gridY) * gridY + me.bottom
      }
    },

    snapToGrid(grid, pendingX, pendingY) {
      const x = Math.round(pendingX / grid[0]) * grid[0];
      const y = Math.round(pendingY / grid[1]) * grid[1];
      return [x, y];
    },

    fireEvent(eventName, eventProperties) {
      const me = this;
      //eventName in ["resizeStart", "resizing", "resizeEnd", "dragStart", dragging", "dragEnd"]
      me.$emit(eventName, {
        x: me.left,
        y: me.top,
        width: me.width,
        height: me.height
      });
    }
  },

  computed: {
    style() {
      const me = this;
      return {
        position: "absolute",
        top: me.top + "px",
        left: me.left + "px",
        width: me.width + "px",
        height: me.height + "px",
        zIndex: me.zIndex,
        ...(me.dragging && me.disableUserSelect ? userSelectNone : userSelectAuto)
      }
    },

    actualHandles() {
      const me = this;
      if (!me.resizable) {
        return [];
      }
      return me.handles;
    },

    width() {
      const me = this;
      return me.parentWidth - me.left - me.right;
    },

    height() {
      const me = this;
      return me.parentHeight - me.top - me.bottom;
    },

    resizingOnX() {
      const me = this;
      return (Boolean(me.handle) && (me.handle.includes("l") || me.handle.includes("r")));
    },

    resizingOnY() {
      const me = this;
      return (Boolean(me.handle) && (me.handle.includes("t") || me.handle.includes("b")));
    },

    isCornerHandle() {
      const me = this;
      return (Boolean(me.handle) && ["tl", "tr", "br", "bl"].includes(me.handle));
    }
  },

  watch: {
    active(val) {
      const me = this;
      me.enabled = val;

      if (val) {
        me.$emit("activated");
      } else {
        me.$emit("deactivated");
      }
    },

    z(val) {
      const me = this;
      if (val >= 0 || val === "auto") {
        me.zIndex = val
      }
    },

    rawLeft(newLeft) {
      const me = this;
      const bounds = me.bounds;

      /*if (bounds !== null) {
        if (newLeft < bounds.minLeft) {
          newLeft = bounds.minLeft;
        } else if (newLeft > bounds.maxLeft) {
          newLeft = bounds.maxLeft;
        }
      }*/
      if (bounds.minLeft !== null && newLeft < bounds.minLeft) {
        newLeft = bounds.minLeft;
      } else if (bounds.maxLeft !== null && bounds.maxLeft < newLeft) {
        newLeft = bounds.maxLeft;
      }

      if (me.lockAspectRatio && me.resizingOnX) {
        me.rawTop = me.top - (me.left - newLeft) / me.aspectFactor;
      }

      me.left = newLeft;
    },

    rawRight(newRight) {
      const me = this;
      const bounds = me.bounds;

      /*if (bounds !== null) {
        if (newRight < bounds.minRight) {
          newRight = bounds.minRight;
        } else if (newRight > bounds.maxRight) {
          newRight = bounds.maxRight;
        }
      }*/
      if (bounds.minRight !== null && newRight < bounds.minRight) {
        newRight = bounds.minRight;
      } else if (bounds.maxRight !== null && bounds.maxRight < newRight) {
        newRight = bounds.maxRight;
      }

      if (me.lockAspectRatio && me.resizingOnX) {
        me.rawBottom = me.bottom - (me.right - newRight) / me.aspectFactor;
      }

      me.right = newRight;
    },

    rawTop(newTop) {
      const me = this;
      const bounds = me.bounds;

      /*if (bounds !== null) {
        if (newTop < bounds.minTop) {
          newTop = bounds.minTop;
        } else if (newTop > bounds.maxTop) {
          newTop = bounds.maxTop;
        }
      }*/
      if (bounds.minTop !== null && newTop < bounds.minTop) {
        newTop = bounds.minTop;
      } else if (bounds.maxTop !== null && bounds.maxTop < newTop) {
        newTop = bounds.maxTop;
      }

      if (me.lockAspectRatio && me.resizingOnY) {
        me.rawLeft = me.left - (me.top - newTop) * me.aspectFactor;
      }

      me.top = newTop;
    },

    rawBottom(newBottom) {
      const me = this;
      const bounds = me.bounds;
      const bottom = me.bottom;

      /*if (bounds !== null) {
        if (newBottom < bounds.minBottom) {
          newBottom = bounds.minBottom;
        } else if (newBottom > bounds.maxBottom) {
          newBottom = bounds.maxBottom;
        }
      }*/
      if (bounds.minBottom !== null && newBottom < bounds.minBottom) {
        newBottom = bounds.minBottom
      } else if (bounds.maxBottom !== null && bounds.maxBottom < newBottom) {
        newBottom = bounds.maxBottom
      }

      if (me.lockAspectRatio && me.resizingOnY) {
        me.rawRight = me.right - (me.bottom - newBottom) * me.aspectFactor
      }

      me.bottom = newBottom;
    },

    x() {
      const me = this;
      if (me.resizing || me.dragging) {
        return false;
      }

      if (me.parent) {
        me.bounds = me.calcDragLimits();
      }

      const delta = me.x - me.left;

      if (delta % me.grid[0] === 0) {
        me.rawLeft = me.x;
        me.rawRight = me.right - delta;
      }
    },

    y() {
      const me = this;
      if (me.resizing || me.dragging) {
        return false;
      }

      if (me.parent) {
        me.bounds = me.calcDragLimits();
      }

      const delta = me.y - me.top;

      if (delta % me.grid[1] === 0) {
        me.rawTop = me.y
        me.rawBottom = me.bottom - delta
      }
    },

    lockAspectRatio(val) {
      const me = this;
      me.aspectFactor = val ? me.width / me.height : undefined;
    },

    minWidth(val) {
      const me = this;
      if (val > 0 && val <= me.width) {
        me.minW = val;
      }
    },

    minHeight(val) {
      const me = this;
      if (val > 0 && val <= me.height) {
        me.minH = val;
      }
    },

    w() {
      const me = this;
      if (me.resizing || me.dragging) {
        return false;
      }

      if (me.parent) {
        me.bounds = me.calcResizeLimits();
      }

      const delta = me.width - me.w;

      if (delta % me.grid[0] === 0) {
        me.rawRight = me.right + delta;
      }
    },

    h() {
      const me = this;
      if (me.resizing || me.dragging) {
        return false;
      }

      if (me.parent) {
        me.bounds = me.calcResizeLimits();
      }

      const delta = me.height - me.h;

      if (delta % me.grid[1] === 0) {
        me.rawBottom = me.bottom + delta;
      }
    }
  }
}
</script>

<style>
.vdr {
  position: absolute;
  box-sizing: border-box;
  border: 1px dashed black;
}

.handle {
  box-sizing: border-box;
  position: absolute;
  width: 10px;
  height: 10px;
  background: #EEE;
  border: 1px solid #333;
}

.handle-tl {
  top: -10px;
  left: -10px;
  cursor: nw-resize;
}

.handle-tm {
  top: -10px;
  left: 50%;
  margin-left: -5px;
  cursor: n-resize;
}

.handle-tr {
  top: -10px;
  right: -10px;
  cursor: ne-resize;
}

.handle-ml {
  top: 50%;
  margin-top: -5px;
  left: -10px;
  cursor: w-resize;
}

.handle-mr {
  top: 50%;
  margin-top: -5px;
  right: -10px;
  cursor: e-resize;
}

.handle-bl {
  bottom: -10px;
  left: -10px;
  cursor: sw-resize;
}

.handle-bm {
  bottom: -10px;
  left: 50%;
  margin-left: -5px;
  cursor: s-resize;
}

.handle-br {
  bottom: -10px;
  right: -10px;
  cursor: se-resize;
}

@media only screen and (max-width: 768px) {
  [class*="handle-"]:before {
    content: '';
    left: -10px;
    right: -10px;
    bottom: -10px;
    top: -10px;
    position: absolute;
  }
}
</style>