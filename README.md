# Inertia Vue Modal POC

Tested with Vue 2.6.12 + Laravel 8.

## Installation

### Client-side installation

In your main JavaScript file, register the `Modalable` and `ToModal` components:

```javascript
import Vue from "vue";
import ToModal from "@/Modal/ToModal";
import Modalable from "@/Modal/Modalable";

Vue.component("Modalable", Modalable);
Vue.component("ToModal", ToModal);
```vue

In your root layout, you need to add the `ComponentModal` as the last component of your template:

```vue
<template>
  <div class="min-h-screen">
    <nav></nav>

    <!-- Page Content -->
    <main>
      <slot />
    </main>

    <ComponentModal />
  </div>
</template>

<script>
import ComponentModal from "@/Modal/ComponentModal";

export default {
  components: {
    ComponentModal,
  },
};
</script>
```

### Server-side installation

In your Laravel application, you only need to add a few lines of code to the `HandleInertiaRequests` middleware.

1. Add the `handle` method to the `HandleInertiaRequests` middleware
2. Add the `isModal` property to the shared data

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Inertia\Inertia;
use Inertia\Middleware;
use Symfony\Component\HttpFoundation\RedirectResponse;

class HandleInertiaRequests extends Middleware
{
    public function handle(Request $request, Closure $next)
    {
        $response = parent::handle($request, $next);

        if ($response instanceof RedirectResponse && (bool) $request->header('X-Inertia-Modal-Redirect-Back')) {
            return back(303);
        }

        if (Inertia::getShared('isModal')) {
            $response->headers->set('X-Inertia-Modal', true);
        }

        return $response;
    }

    public function share(Request $request)
    {
        return array_merge(parent::share($request), [
            'isModal' => (bool) $request->header('X-Inertia-Modal'),
        ]);
    }
}
```

## Usage

Since we added the `ComponentModal` component, the global `$inertia` object now has a `visitInModal` method. This allows you to make an Inertia visit that loads into the modal.

You can use this method, for example, in the `@click` handler of a button:

```vue
<button @click="$inertia.visitInModal('/user/create')">Load in modal</button>
```

Instead of using the method in your template, you can also use it in your script:

```vue
<script>
export default {
  methods: {
    openModal() {
      this.$inertia.visitInModal('/user/create');
    },
  },
};
</script>
```

In most cases, the `/user/create` endpoint renders a form that's wrapped into a template, maybe with some additional components and components to style the form within your template. Here's a simple example of what the `UserCreate.vue` component might look like:

```vue
<template>
  <!-- app-layout provides the sidebar navigation and footer -->
  <app-layout>
    <!-- form-panel provides a nice padding with padding and shadow -->
    <form-panel>
      <form @submit.prevent="form.post('/login')">
        <!-- name -->
        <input type="text" v-model="form.name">
        <div v-if="form.errors.name">{{ form.errors.name }}</div>

        <!-- email -->
        <input type="email" v-model="form.email">
        <div v-if="form.errors.email">{{ form.errors.email }}</div>

        <!-- submit -->
        <button type="submit" :disabled="form.processing">Login</button>
      </form>
    </form-panel>
  </app-layout>
</template>

<script>
export default {
  data() {
    return {
      form: this.$inertia.form({
        name: "",
        email: "",
      }),
    };
  },
};
</script>
```

To load this form into a modal, we don't want the sidear, footer and styling form the `form-panel` component. We want just the `form` itself!

To accomplish this, we need to wrap the whole template into the `Modalable` component. The `is-modal` prop value comes from the `IsModalable` mixin that we need to add as well.

The last step is to move the `form` out of the `app-layout` by using the `ToModal` component and `#toModal` template. The template goes below the `app-layout` (but still in `Modalable`!) and holds the form itself. The contents of this template will be shown in the modal. In the `app-layout` we need to add a `ToModal` component at the original position of the form. This way, the form is still at the right place when we visit this route *without* loading it into a modal.

```vue
<template>
  <Modalable :is-modal="isModal">
    <app-layout>
      <form-panel>
        <ToModal />
      </form-panel>
    </app-layout>

    <template #toModal>
      <form @submit.prevent="form.post('/login')">
        <!-- name -->
        <input type="text" v-model="form.name">
        <div v-if="form.errors.name">{{ form.errors.name }}</div>

        <!-- email -->
        <input type="email" v-model="form.email">
        <div v-if="form.errors.email">{{ form.errors.email }}</div>

        <!-- submit -->
        <button type="submit" :disabled="form.processing">Login</button>
        </form>
    </template>
  </Modalable>
</template>

<script>
import IsModalable from "@/Modal/IsModalable";

export default {
  mixins: [IsModalable],

  data() {
    return {
      form: this.$inertia.form({
        name: "",
        email: "",
      }),
    };
  },
};
</script>
```