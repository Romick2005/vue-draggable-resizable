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
      validator: (val) => val > 0
    },

    h: {
      type: Number,
      default: 200,
      validator: (val) => val > 0
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
    return {
      rawWidth: me.w,
      rawHeight: me.h,
      rawLeft: me.x,
      rawTop: me.y,
      rawRight: null,
      rawBottom: null,

      left: me.x,
      top: me.y,
      right: null,
      bottom: null,

      aspectFactor: me.w / me.h,

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
    addEvent(documentElement, "mousedown", me.deselect);
    addEvent(documentElement, "touchend touchcancel", me.deselect);
    addEvent(window, "resize", me.checkParentSize);
  },

  beforeDestroy() {
    const me = this;
    const documentElement = document.documentElement;
    removeEvent(documentElement, "mousedown", me.deselect);
    removeEvent(documentElement, "touchstart", me.handleUp);
    removeEvent(documentElement, "mousemove", me.move);
    removeEvent(documentElement, "touchmove", me.move);
    removeEvent(documentElement, "mouseup", me.handleUp);
    removeEvent(documentElement, "touchend touchcancel", me.deselect);
    removeEvent(window, "resize", me.checkParentSize);
  },

  methods: {
    resetBoundsAndMouseState() {
      const me = this;
      me.mouseClickPosition = {
        mouseX: 0,
        mouseY: 0,
        left: 0,
        top: 0,
        w: 0,
        h: 0
      };

      me.bounds = {
        minLeft: null,
        maxLeft: null,
        minRight: null,
        maxRight: null,
        minTop: null,
        maxTop: null,
        minBottom: null,
        maxBottom: null
      }
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

        if (this.draggable) {
          me.dragging = true;
        }

        const mouseClickPosition = me.mouseClickPosition;
        const touches = e.touches;
        mouseClickPosition.mouseX = touches ? touches[0].pageX : e.pageX;
        mouseClickPosition.mouseY = touches ? touches[0].pageY : e.pageY;
        mouseClickPosition.left = me.left;
        mouseClickPosition.right = me.right;
        mouseClickPosition.top = me.top;
        mouseClickPosition.bottom = me.bottom;

        if (me.parent) {
          me.bounds = me.calcDragLimits();
        }

        const documentElement = document.documentElement;
        addEvent(documentElement, eventsFor.move, this.move);
        addEvent(documentElement, eventsFor.stop, this.handleUp);
      }
    },

    calcDragLimits() {
      const me = this;
      return {
        minLeft: (me.parentWidth + me.left) % me.grid[0],
        maxLeft: Math.floor((me.parentWidth - me.width - me.left) / me.grid[0]) * me.grid[0] + me.left,
        minRight: (me.parentWidth + me.right) % me.grid[0],
        maxRight: Math.floor((me.parentWidth - me.width - me.right) / me.grid[0]) * me.grid[0] + me.right,
        minTop: (me.parentHeight + me.top) % me.grid[1],
        maxTop: Math.floor((me.parentHeight - me.height - me.top) / me.grid[1]) * me.grid[1] + me.top,
        minBottom: (me.parentHeight + me.bottom) % me.grid[1],
        maxBottom: Math.floor((me.parentHeight - me.height - me.bottom) / me.grid[1]) * me.grid[1] + me.bottom
      }
    },

    deselect(e) {
      const me = this;
      const target = e.target || e.srcElement;
      const regex = new RegExp("vdr" + "-([trmbl]{2})", "");

      if (!me.$el.contains(target) && !regex.test(target.className)) {
        if (me.enabled && !this.preventDeactivation) {
          me.enabled = false;
          me.$emit("deactivated");
          me.$emit("update:active", false);
        }

        removeEvent(document.documentElement, eventsFor.move, me.handleMove);
      }

      me.resetBoundsAndMouseState();
    },

    handleTouchDown(handle, e) {
      const me = this;
      eventsFor = events.touch;
      me.handleDown(handle, e);
    },

    handleDown(handle, e) {
      if (e.stopPropagation) {
        e.stopPropagation();
      }
      const me = this;
      // Here we avoid a dangerous recursion by faking
      // corner handles as middle handles
      if (me.lockAspectRatio && !handle.includes("m")) {
        me.handle = "m" + handle.substring(1);
      } else {
        me.handle = handle;
      }

      me.resizing = true;

      const mouseClickPosition = me.mouseClickPosition;
      mouseClickPosition.mouseX = e.touches ? e.touches[0].pageX : e.pageX;
      mouseClickPosition.mouseY = e.touches ? e.touches[0].pageY : e.pageY;
      mouseClickPosition.left = me.left;
      mouseClickPosition.right = me.right;
      mouseClickPosition.top = me.top;
      mouseClickPosition.bottom = me.bottom;

      me.bounds = me.calcResizeLimits();

      addEvent(document.documentElement, eventsFor.move, me.handleMove);
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
        me.fireEvent("resizestop");
      }
      if (me.dragging) {
        me.dragging = false;
        const mouseClickPosition = me.mouseClickPosition;

        // Check if there was any dragging changes
        if (mouseClickPosition.left !== me.left || mouseClickPosition.top !== me.top) {
          me.fireEvent("dragstop");
        }
      }
      me.resetBoundsAndMouseState();
      removeEvent(document.documentElement, eventsFor.move, me.handleMove);
    },

    snapToGrid(grid, pendingX, pendingY) {
      const x = Math.round(pendingX / grid[0]) * grid[0];
      const y = Math.round(pendingY / grid[1]) * grid[1];
      return [x, y];
    },

    fireEvent(eventName, eventProperties) {
      const me = this;
      //eventName in ["resizestop", "dragstop", "dragging", "resizing"]
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
      const aspectFactor = me.aspectFactor;
      const lockAspectRatio = me.lockAspectRatio;
      const left = me.left;
      const top = me.top;

      if (bounds.minLeft !== null && newLeft < bounds.minLeft) {
        newLeft = bounds.minLeft;
      } else if (bounds.maxLeft !== null && bounds.maxLeft < newLeft) {
        newLeft = bounds.maxLeft;
      }

      if (lockAspectRatio && me.resizingOnX) {
        me.rawTop = top - (left - newLeft) / aspectFactor;
      }

      me.left = newLeft;
    },

    rawRight(newRight) {
      const me = this;
      const bounds = me.bounds;
      const aspectFactor = me.aspectFactor;
      const lockAspectRatio = me.lockAspectRatio;
      const right = me.right;
      const bottom = me.bottom;

      if (bounds.minRight !== null && newRight < bounds.minRight) {
        newRight = bounds.minRight;
      } else if (bounds.maxRight !== null && bounds.maxRight < newRight) {
        newRight = bounds.maxRight;
      }

      if (lockAspectRatio && me.resizingOnX) {
        me.rawBottom = bottom - (right - newRight) / aspectFactor;
      }

      me.right = newRight;
    },

    rawTop(newTop) {
      const me = this;
      const bounds = me.bounds;
      const aspectFactor = me.aspectFactor;
      const lockAspectRatio = me.lockAspectRatio;
      const left = me.left;
      const top = me.top;

      if (bounds.minTop !== null && newTop < bounds.minTop) {
        newTop = bounds.minTop;
      } else if (bounds.maxTop !== null && bounds.maxTop < newTop) {
        newTop = bounds.maxTop;
      }

      if (lockAspectRatio && me.resizingOnY) {
        me.rawLeft = left - (top - newTop) * aspectFactor;
      }

      me.top = newTop;
    },

    rawBottom(newBottom) {
      const me = this;
      const bounds = me.bounds;
      const aspectFactor = me.aspectFactor;
      const lockAspectRatio = me.lockAspectRatio;
      const right = me.right;
      const bottom = me.bottom;

      if (bounds.minBottom !== null && newBottom < bounds.minBottom) {
        newBottom = bounds.minBottom
      } else if (bounds.maxBottom !== null && bounds.maxBottom < newBottom) {
        newBottom = bounds.maxBottom
      }

      if (lockAspectRatio && me.resizingOnY) {
        me.rawRight = right - (bottom - newBottom) * aspectFactor
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
