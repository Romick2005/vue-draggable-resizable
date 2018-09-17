<template>
  <div
    :style="style"
    :class="{
      draggable: draggable,
      resizable: resizable,
      active: enabled,
      dragging: dragging,
      resizing: resizing}"
    class="vdr"
    @mousedown.stop="elmDown"
    @touchstart.stop="elmDown"
    @dblclick="fillParent">
    <div
      v-for="handle in handles"
      v-if="resizable"
      :key="handle"
      :class="'handle-' + handle"
      :style="{ display: enabled ? 'block' : 'none'}"
      class="handle"
      @mousedown.stop.prevent="handleDown(handle, $event)"
      @touchstart.stop.prevent="handleDown(handle, $event)"/>
    <slot/>
  </div>
</template>

<script>
import { matchesSelectorToParentElements } from "../utils";

export default {
  replace: true,
  name: "VueDraggableResizable",

  props: {
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

    w: {
      type: Number,
      default: 200,
      validator(val) {
        return val > 0;
      }
    },

    h: {
      type: Number,
      default: 200,
      validator(val) {
        return val > 0;
      }
    },

    minw: {
      type: Number,
      default: 50,
      validator(val) {
        return val >= 0;
      }
    },

    minh: {
      type: Number,
      default: 50,
      validator(val) {
        return val >= 0;
      }
    },

    x: {
      type: Number,
      default: 0,
      validator(val) {
        return typeof val === "number";
      }
    },

    y: {
      type: Number,
      default: 0,
      validator(val) {
        return typeof val === "number";
      }
    },

    z: {
      type: [ String, Number ],
      default: "auto",
      validator(val) {
        let valid = (typeof val === "string") ? val === "auto" : val >= 0;
        return valid;
      }
    },

    handles: {
      type: Array,
      default() {
        return ["tl", "tm", "tr", "mr", "br", "bm", "bl", "ml"];
      },
      validator(val) {
        var s = new Set(["tl", "tm", "tr", "mr", "br", "bm", "bl", "ml"]);
        return new Set(val.filter(h => s.has(h))).size === val.length;
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
      validator(val) {
        return ["x", "y", "both"].indexOf(val) !== -1;
      }
    },

    grid: {
      type: Array,
      default() {
        return [1, 1];
      }
    },

    parent: {
      type: Boolean,
      default: false
    },

    maximize: {
      type: Boolean,
      default: false
    }
  },

  created() {
    const me = this;
    me.parentX = 0;
    me.parentW = 9999;
    me.parentY = 0;
    me.parentH = 9999;
    me.mouseX = 0;
    me.mouseY = 0;
    me.lastMouseX = 0;
    me.lastMouseY = 0;
    me.mouseOffX = 0;
    me.mouseOffY = 0;
    me.elmX = 0;
    me.elmY = 0;
    me.elmW = 0;
    me.elmH = 0;
  },

  mounted() {
    const me = this;
    const documentElement = document.documentElement;
    documentElement.addEventListener("mousemove", me.handleMove, true);
    documentElement.addEventListener("mousedown", me.deselect, true);
    documentElement.addEventListener("mouseup", me.handleUp, true);

    // touch events bindings
    documentElement.addEventListener("touchmove", me.handleMove, true);
    documentElement.addEventListener("touchend touchcancel", me.deselect, true);
    documentElement.addEventListener("touchstart", me.handleUp, true);
    const el = me.$el;
    const elStyle = el.style;
    me.elmX = parseInt(elStyle.left, 10);
    me.elmY = parseInt(elStyle.top, 10);
    me.elmW = el.offsetWidth || el.clientWidth;
    me.elmH = el.offsetHeight || el.clientHeight;
    me.reviewDimensions();
  },

  beforeDestroy() {
    const me = this;
    const documentElement = document.documentElement;
    documentElement.removeEventListener("mousemove", me.handleMove, true);
    documentElement.removeEventListener("mousedown", me.deselect, true);
    documentElement.removeEventListener("mouseup", me.handleUp, true);

    // touch events bindings removed
    documentElement.addEventListener("touchmove", me.handleMove, true);
    documentElement.addEventListener("touchend touchcancel", me.deselect, true);
    documentElement.addEventListener("touchstart", me.handleUp, true);
  },

  data() {
    const me = this;
    return {
      top: me.y,
      left: me.x,
      width: me.w,
      height: me.h,
      resizing: false,
      dragging: false,
      enabled: me.active,
      handle: null,
      zIndex: me.z
    };
  },

  methods: {
    reviewDimensions() {
      const me = this;
      if (me.w < me.minw) {
        me.width = me.minw;
      }
      if (me.h < me.minh) {
        me.height = me.minh;
      }
      if (me.parent) {
        const parentNode = me.$el.parentNode;
        const parentW = parseInt(parentNode.clientWidth, 10);
        const parentH = parseInt(parentNode.clientHeight, 10);
        me.parentW = parentW;
        me.parentH = parentH;
        if (me.w > me.parentW) {
          me.width = parentW;
        }
        if (me.h > me.parentH) {
          me.height = parentH;
        }
        if (me.x + me.w > me.parentW) {
          me.width = parentW - me.x;
        }
        if (me.y + me.h > me.parentH) {
          me.height = parentH - me.y;
        }
      }
      me.elmW = me.width;
      me.elmH = me.height;
      me.$emit("resizing", me.left, me.top, me.width, me.height);
    },

    elmDown(e) {
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
        e.preventDefault();
        me.reviewDimensions();
        if (!me.enabled) {
          me.enabled = true;
          me.$emit("activated");
          me.$emit("update:active", true);
        }
        if (me.draggable) {
          me.dragging = true;
        }
      }
    },

    deselect(e) {
      const me = this;
      if (e.type.indexOf("touch") !== -1) {
        const touches = e.changedTouches[0];
        me.mouseX = touches.clientX;
        me.mouseY = touches.clientY;
      } else {
        const documentElement = document.documentElement;
        me.mouseX = e.pageX || e.clientX + documentElement.scrollLeft;
        me.mouseY = e.pageY || e.clientY + documentElement.scrollTop;
      }
      me.lastMouseX = me.mouseX;
      me.lastMouseY = me.mouseY;
      const target = e.target || e.srcElement;
      const regex = new RegExp("handle-([trmbl]{2})", "");
      if (!me.$el.contains(target) && !regex.test(target.className)) {
        if (me.enabled) {
          me.enabled = false;
          me.$emit("deactivated");
          me.$emit("update:active", false);
        }
      }
    },

    handleDown(handle, e) {
      const me = this;
      me.handle = handle;
      if (e.stopPropagation) {
        e.stopPropagation();
      }
      if (e.preventDefault) {
        e.preventDefault();
      }
      me.resizing = true;
    },

    fillParent(e) {
      const me = this;
      if (!me.parent || !me.resizable || !me.maximize) {
        return false;
      }
      let done = false;
      const animate = function () {
        if (!done) {
          window.requestAnimationFrame(animate);
        }
        if (me.axis === "x") {
          if (me.width === me.parentW && me.left === me.parentX) {
            done = true;
          }
        } else if (me.axis === "y") {
          if (me.height === me.parentH && me.top === me.parentY) {
            done = true;
          }
        } else if (me.axis === "both") {
          if (
            me.width === me.parentW &&
            me.height === me.parentH &&
            me.top === me.parentY &&
            me.left === me.parentX
          ) {
            done = true;
          }
        }

        if (me.axis === "x" || me.axis === "both") {
          if (me.width < me.parentW) {
            me.width++;
            me.elmW++;
          }
          if (me.left > me.parentX) {
            me.left--;
            me.elmX--;
          }
        }
        if (me.axis === "y" || me.axis === "both") {
          if (me.height < me.parentH) {
            me.height++;
            me.elmH++;
          }
          if (me.top > me.parentY) {
            me.top--;
            me.elmY--;
          }
        }
        me.$emit("resizing", me.left, me.top, me.width, me.height);
      }
      window.requestAnimationFrame(animate);
    },

    handleMove(e) {
      const me = this;
      const documentElement = document.documentElement;
      const isTouchMove = e.type.indexOf("touchmove") !== -1;
      const touches = e.touches[0];
      me.mouseX = isTouchMove
        ? touches.clientX
        : e.pageX || e.clientX + documentElement.scrollLeft;
      me.mouseY = isTouchMove
        ? touches.clientY
        : e.pageY || e.clientY + documentElement.scrollTop;

      let diffX = me.mouseX - me.lastMouseX + me.mouseOffX;
      let diffY = me.mouseY - me.lastMouseY + me.mouseOffY;
      me.mouseOffX = me.mouseOffY = 0;
      me.lastMouseX = me.mouseX;
      me.lastMouseY = me.mouseY;
      let dX = diffX;
      let dY = diffY;
      if (me.resizing) {
        if (me.handle.indexOf("t") >= 0) {
          if (me.elmH - dY < me.minh) {
            me.mouseOffY = (dY - (diffY = me.elmH - me.minh));
          } else if (me.parent && me.elmY + dY < me.parentY) {
            me.mouseOffY = (dY - (diffY = me.parentY - me.elmY));
          }
          me.elmY += diffY;
          me.elmH -= diffY;
        }
        if (me.handle.indexOf("b") >= 0) {
          if (me.elmH + dY < me.minh) {
            me.mouseOffY = (dY - (diffY = me.minh - me.elmH));
          } else if (me.parent && me.elmY + me.elmH + dY > me.parentH) {
            me.mouseOffY = (dY - (diffY = me.parentH - me.elmY - me.elmH));
          }
          me.elmH += diffY;
        }
        if (me.handle.indexOf("l") >= 0) {
          if (me.elmW - dX < me.minw) {
            me.mouseOffX = (dX - (diffX = me.elmW - me.minw));
          } else if (me.parent && me.elmX + dX < me.parentX) {
            me.mouseOffX = (dX - (diffX = me.parentX - me.elmX));
          }
          me.elmX += diffX;
          me.elmW -= diffX;
        }
        if (me.handle.indexOf("r") >= 0) {
          if (me.elmW + dX < me.minw) {
            me.mouseOffX = dX - (diffX = me.minw - me.elmW);
          } else if (me.parent && me.elmX + me.elmW + dX > me.parentW) {
            me.mouseOffX = dX - (diffX = me.parentW - me.elmX - me.elmW);
          }
          me.elmW += diffX;
        }
        me.left = (Math.round(me.elmX / me.grid[0]) * me.grid[0]);
        me.top = (Math.round(me.elmY / me.grid[1]) * me.grid[1]);
        me.width = (Math.round(me.elmW / me.grid[0]) * me.grid[0]);
        me.height = (Math.round(me.elmH / me.grid[1]) * me.grid[1]);
        me.$emit("resizing", me.left, me.top, me.width, me.height);
      } else if (me.dragging) {
        if (me.parent) {
          if (me.elmX + dX < me.parentX) {
            me.mouseOffX = dX - (diffX = me.parentX - me.elmX);
          } else if (me.elmX + me.elmW + dX > me.parentW) {
            me.mouseOffX = dX - (diffX = me.parentW - me.elmX - me.elmW);
          }
          if (me.elmY + dY < me.parentY) {
            me.mouseOffY = dY - (diffY = me.parentY - me.elmY);
          } else if (me.elmY + me.elmH + dY > me.parentH) {
            me.mouseOffY = dY - (diffY = me.parentH - me.elmY - me.elmH);
          }
        }
        me.elmX += diffX;
        me.elmY += diffY;
        if (me.axis === "x" || me.axis === "both") {
          me.left = (Math.round(me.elmX / me.grid[0]) * me.grid[0]);
        }
        if (me.axis === "y" || me.axis === "both") {
          me.top = (Math.round(me.elmY / me.grid[1]) * me.grid[1]);
        }
        me.$emit("dragging", me.left, me.top);
      }
    },

    handleUp(e) {
      const me = this;
      if (e.type.indexOf("touch") !== -1) {
        me.lastMouseX = e.changedTouches[0].clientX;
        me.lastMouseY = e.changedTouches[0].clientY;
      }
      me.handle = null;
      if (me.resizing) {
        me.resizing = false;
        me.$emit("resizestop", me.left, me.top, me.width, me.height);
      }
      if (me.dragging) {
        me.dragging = false;
        me.$emit("dragstop", me.left, me.top);
      }
      me.elmX = me.left;
      me.elmY = me.top;
    }
  },

  computed: {
    style() {
      const me = this;
      return {
        top: me.top + "px",
        left: me.left + "px",
        width: me.width + "px",
        height: me.height + "px",
        zIndex: me.zIndex
      };
    }
  },

  watch: {
    active(val) {
      const me = this;
      me.enabled = val;
    },

    z(val) {
      const me = this;
      if (val >= 0 || val === "auto") {
        me.zIndex = val;
      }
    }
  }
}
</script>

<style>
.vdr {
  position: absolute;
  box-sizing: border-box;
}

.handle {
  box-sizing: border-box;
  display: none;
  position: absolute;
  width: 10px;
  height: 10px;
  font-size: 1px;
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
  /* For mobile phones: */
  [class*="handle-"]:before {
    content: "";
    left: -10px;
    right: -10px;
    bottom: -10px;
    top: -10px;
    position: absolute;
  }
}
</style>
