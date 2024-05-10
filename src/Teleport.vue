<template>
  <div :class="classes">
    <slot/>
  </div>
</template>

<script>
export default {
  name: 'teleport',
  props: {
    to: {
      type: String,
      required: true,
    },
    where: {
      type: String,
      default: 'after',
    },
    disabled: Boolean,
  },
  data() {
    this.nodes = [];
    this.observer = null;
    this.parent = null;
    return {}
  },
  watch: {
    to: 'maybeMove',
    where: 'maybeMove',
    disabled(value) {
      if (value) {
        this.disable();
        // Ensure all event done.
        this.$nextTick(() => {
          this.teardownObserver();
        });
      } else {
        this.bootObserver();
        this.move();
      }
    },
  },
  mounted() {
    // Store a reference to the nodes
    this.nodes = Array.from(this.$el.childNodes);

    if (!this.disabled) {
      this.bootObserver();
    }

    // Move slot content to target
    this.maybeMove();
  },
  beforeDestroy() {
    // Fix node reference
    this.nodes = this.getComponentChildrenNode();

    // Move back
    this.disable();

    // Stop observing
    this.teardownObserver();
  },
  computed: {
    classes() {
      if (this.disabled) {
        return ['teleporter'];
      }

      return ['teleporter', 'hidden'];
    },
  },
  methods: {
    maybeMove() {
      if (!this.disabled) {
        this.move();
      }
    },
    move() {
      this.parent = document.querySelector(this.to);
      if (!this.parent) {
        this.disable();
        return;
      }

      if (this.where === 'before') {
        this.parent.prepend(this.getFragment());
      } else {
        this.parent.appendChild(this.getFragment());
      }
    },
    disable() {
      this.$el.appendChild(this.getFragment());
      this.parent = null;
    },
    // Using a fragment is faster because it'll trigger only a single reflow
    // See https://developer.mozilla.org/en-US/docs/Web/API/DocumentFragment
    getFragment() {
      const fragment = document.createDocumentFragment();

      this.nodes.forEach(node => fragment.appendChild(node));

      return fragment;
    },
    bootObserver() {
      if (this.observer) {
        return;
      }
      // watch childNodes change
      this.observer = new MutationObserver((mutations) => {
        // Remove old nodes before update position.
        this.getFragment();
        this.nodes = this.getComponentChildrenNode();
        this.maybeMove();
      });

      this.observer.observe(this.$el, {
        childList: true,
        subtree: false,
        attributes: false,
        characterData: false,
      });
    },
    teardownObserver() {
      if (this.observer) {
        this.observer.disconnect();
        this.observer = null;
      }
    },
    getComponentChildrenNode() {
      return this.$vnode.componentOptions.children.map((i) => i.elm).filter((i) => i);
    },
  },
};
</script>

<style scoped lang="scss">
.hidden {
  visibility: hidden;
  display: none;
}
</style>
