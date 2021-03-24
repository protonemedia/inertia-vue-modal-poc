# Inertia Vue Modal POC

Tested with Vue 2.6.12 + Laravel 8.

I've copied the default [Laravel Jetstream](https://jetstream.laravel.com/2.x/stacks/inertia.html) modal component for this demo.

## Installation

### Client-side installation

```bash
npm i @protonemedia/inertia-vue-modal-poc
```

The only dependency for this POC to work is [vue-portal](https://github.com/LinusBorg/portal-vue).

In your main JavaScript file, register the `Modalable` and `ToModal` components:

```javascript
import Vue from "vue";
import { Modalable, ToModal } from "@protonemedia/inertia-vue-modal-poc"
import PortalVue from "portal-vue";

Vue.component("Modalable", Modalable);
Vue.component("ToModal", ToModal);
Vue.use(PortalVue)
```

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
import { ComponentModal } from "@protonemedia/inertia-vue-modal-poc"

export default {
  components: {
    ComponentModal,
  },
};
</script>
```

### Server-side installation

In your Laravel application, you only need to add a few lines of code to the `HandleInertiaRequests` middleware.

1. Add the `handle` method to the `HandleInertiaRequests` middleware.
2. Add the `isModal` property to the shared data array.

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

### Update the page you want to load into a modal

In most cases, the `/user/create` endpoint renders a form that's wrapped into a template, maybe with other components and components to style the form. Here's a simple example of what the `UserCreate.vue` component might look like:

```vue
<template>
  <!-- app-layout provides the sidebar navigation and footer -->
  <app-layout>
    <!-- form-panel provides a nice padding with padding and shadow -->
    <form-panel>
      <form @submit.prevent="form.post('/user/store')">
        <input type="text" v-model="form.name">
        <input type="email" v-model="form.email">

        <button type="submit">Login</button>
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

To load this form into a modal, we don't want the sidebar, footer, and styling from the `form-panel` component. We want just the `form` itself!

To accomplish this, you need to do three things:

1. Add the `IsModalable` mixin to your component.
2. Wrap your *whole* component into the `Modalable` component.
3. Move the `form` to a separate `#toModal` template and replace it with a `ToModal` component.

```vue
<template>
  <Modalable :is-modal="isModal">
    <app-layout>
      <form-panel>
        <ToModal />
      </form-panel>
    </app-layout>

    <template #toModal>
      <form @submit.prevent="form.post('/user.store')">
        <input type="text" v-model="form.name">
        <input type="email" v-model="form.email">

        <!-- submit -->
        <button type="submit">Login</button>
      </form>
    </template>
  </Modalable>
</template>

<script>
import { Modalable, IsModalable } from "@protonemedia/inertia-vue-modal-poc"

export default {
  mixins: [IsModalable],

  components: {
    Modalable
  },

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

Now when you visit `/user/create`, nothing has changed! You still have your layout and `form-panel` styling. But when you load this component into a modal, it will only render the `form`.

### Handling redirects

By default, redirects are handled as any other Inertia request. For example you're visiting `/user`, you open `/user/create` in a modal, and after a successful submit, you redirect the user to the detail page of the newly created user:

```php
public function store(UserStoreRequest $request)
{
    $user = User::create(...);

    return redirect()->route('user.show', $user);
}
```

You might not always want to route to the detail page. Luckily, you don't have to update your server-side implementation.

The `visitInModal` method accepts a second argument that can either a Boolean or a callback. Instead of redirecting the user, the user stays on the same page, and you can manually handle the event with the callback. This callback is executed after a successful request, for example, when the new user is stored in the database.

```javascript
this.$inertia.visitInModal('/user/create', (event) => {
  // do something
});
```
