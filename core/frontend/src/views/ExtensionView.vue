<template>
  <BrIframe
    v-if="detected_port"
    :source="service_path"
    :class="fullpage ? 'fullpage' : ''"
  />
</template>

<script lang="ts">
import Vue from 'vue'

import BrIframe from '@/components/utils/BrIframe.vue'
import services_scanner from '@/store/servicesScanner'

export default Vue.extend({
  name: 'ExtensionView',
  components: {
    BrIframe,
  },
  data() {
    return {
      detected_port: undefined as number | undefined,
    }
  },
  computed: {
    fullpage(): boolean {
      return this.$route.query.full_page === 'true'
    },
    port(): number {
      return services_scanner.services.filter(
        (service) => service?.metadata?.sanitized_name === this.$route.params.name,
      )[0]?.port ?? 80
    },
    service_path(): string {
      return `${window.location.protocol}//${window.location.hostname}:${this.detected_port}`
    },
  },
  watch: {
    port(new_value) {
      if (new_value !== undefined) {
        this.detected_port = new_value
      }
    },
  },
})
</script>

<style>
.fullpage {
  position:absolute; left: 0; right: 0; bottom: 0; top: 0px;
}
</style>
