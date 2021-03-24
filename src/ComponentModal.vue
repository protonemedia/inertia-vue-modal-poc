<template>
  <JetstreamModal :show="modal != null" @close="close">
    <component v-if="modal" :is="modal.component" v-bind="modal.page.props" />
  </JetstreamModal>
</template>

<script>
import JetstreamModal from "./JetstreamModal";
import Axios from "axios";
import {
  hrefToUrl,
  urlWithoutHash,
  mergeDataIntoQueryString,
} from "@inertiajs/inertia";

export default {
  components: { JetstreamModal },

  data() {
    return {
      modal: null,
    };
  },

  beforeMount() {
    this.$inertia.visitInModal = (url, onSuccess) => {
      this.visitInModal(url, onSuccess);
    };

    this.$inertia.on("success", (event) => {
      if (typeof this.modal?.onSuccess === "function") {
        this.modal.onSuccess(event);
      }

      this.close();
    });
  },

  methods: {
    close() {
      if (this.modal) {
        // remove the 'X-Inertia-Modal' and 'X-Inertia-Modal-Redirect-Back' headers for future requests
        this.modal.removeBeforeEventListener();
      }

      this.modal = null;
    },

    visitInModal(url, onSuccess) {
      let data = {};

      [url, data] = mergeDataIntoQueryString("get", hrefToUrl(url), {});

      Axios({
        method: "get",
        url: urlWithoutHash(url).href,
        data: {},
        params: data,
        headers: {
          Accept: "text/html, application/xhtml+xml",
          "X-Requested-With": "XMLHttpRequest",
          "X-Inertia": true,
          "X-Inertia-Modal": true,
          "X-Inertia-Version": this.$page.version,
        },
      }).then((response) => {
        const Inertia = this.$inertia;
        const page = response.data;

        return Promise.resolve(Inertia.resolveComponent(page.component)).then(
          (component) => {
            const clone = JSON.parse(JSON.stringify(page));
            clone.props = Inertia.transformProps(clone.props);

            const removeBeforeEventListener = Inertia.on("before", (event) => {
              // make sure the backend knows we're requesting from within a modal
              event.detail.visit.headers["X-Inertia-Modal"] = true;

              if (onSuccess) {
                event.detail.visit.headers[
                  "X-Inertia-Modal-Redirect-Back"
                ] = true;
              }
            });

            this.modal = {
              component,
              onSuccess,
              removeBeforeEventListener,
              page: clone,
            };
          }
        );
      });
    },
  },
};
</script>