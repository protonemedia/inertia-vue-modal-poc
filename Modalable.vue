<template>
  <div>
    <portal-target v-if="isModal" :name="portalName" />
    <slot v-else></slot>

    <portal :to="portalName">
      <slot name="toModal" />
    </portal>
  </div>
</template>

<script>
export default {
  props: {
    isModal: {
      type: Boolean,
      default: false,
    },

    portalName: {
      type: String,
      default: "visit-modal",
      required: false,
    },
  },
};
</script>