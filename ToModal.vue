<template>
  <portal-target :name="portalName" />
</template>

<script>
export default {
  props: {
    portalName: {
      type: String,
      default: "visit-modal",
      required: false,
    },
  },
};
</script>