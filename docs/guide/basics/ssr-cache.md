# SSR Cache

Vue Storefront generates the server-side rendered pages to improve SEO results. In the latest version of Vue Storefront, we added the output cache option (disabled by default) to improve performance.

The output cache is set by the following `config/local.json` variables:

```json
    "server": {
      "host": "localhost",
      "port": 3000,
      "protocol": "http",
      "api": "api",
      "useOutputCacheTagging": true,
      "useOutputCache": true,
      "outputCacheDefaultTtl": 86400
    },
    "redis": {
      "host": "localhost",
      "port": 6379,
      "db": 0
    },
```

## Dynamic tags

The dynamic tags config option: `useOutputCacheTagging` - if set to `true`, Vue Storefront is generating the special HTTP Header `X-VS-Cache-Tags`

```js
res.setHeader('X-VS-Cache-Tags', cacheTags);
```

Cache tags are assigned regarding the products and categories that are used on the specific page. A typical `X-VS-Cache-Tags` tag looks like this:

```
X-VS-Cache-Tags: P1852 P198 C20
```

The tags can be used to invalidate the Varnish cache, if you're using it. [Read more on that](https://www.drupal.org/docs/8/api/cache-api/cache-tags-varnish).

## Redis

If both `useOutputCache` and `useOutputCacheTagging` options are set to `true`, Vue Storefront is using output cache stored in Redis (configured in the redis section of the config file). Cache is tagged with dynamic tags and can be invalidated using a special webhook:

An example call to clear all pages containing specific product and category:

```bash
curl http://localhost:8000/invalidate?tag=P1852,C20
```

An example call to clear all product, category, and homepages:

```bash
curl http://localhost:8000/invalidate?tag=product,category,home
```

:::warning Caution !
We strongly recommend you DO NOT USE output cache in development mode. By using it, you won't be able to refresh the UI changes after modifying the Vue components, etc.
:::

## CLI cache clear

You can manually clear the Redis cache for specific tags by running the following command:

```bash
npm run cache clear
npm run cache clear -- --tag=product,category
npm run cache clear -- --tag=P198
npm run cache clear -- --tag=*
```

Available tag keys are set in the `config.server.availableCacheTags` (by default: `"product", "category", "home", "checkout", "page-not-found", "compare", "my-account", "P", "C"`)

**Dynamic cache invalidation:** Recent version of [mage2vuestorefront](https://github.com/DivanteLtd/mage2vuestorefront) supports output cache invalidation. Output cache is tagged with the product and categories ID (products and categories used on a specific page). Mage2vuestorefront can invalidate the cache of a product and category pages if you set the following ENV variables:

```bash
export VS_INVALIDATE_CACHE_URL=http://localhost:8000/invalidate?key=SECRETKEY&tag=
export VS_INVALIDATE_CACHE=1
```

:::warning Security note
Please note that `key=SECRETKEY` should be equal to `vue-storefront/config/local.json` value of `server.invalidateCacheKey`
:::

## Adding new types / cache tags

If you're adding a new type of page (`core/pages`) and `config.server.useOutputCache=true`, you should also extend the `config.server.availableCacheTags` of new general purpose tag that will be connected with the URLs connected with this new page.

After doing so, please add the `asyncData` method to your page code assigning the right tag (please take a look at `core/pages/Home.js` for instance):

```js
  asyncData ({ store, route, context }) { // this is for SSR purposes to prefetch data
    return new Promise((resolve, reject) => {
      if (context) context.output.cacheTags.add(`home`)
      Logger.log('Entering asyncData for Home root ' + new Date())()
      EventBus.$emitFilter('home-after-load', { store: store, route: route }).then((results) => {
        return resolve()
      }).catch((err) => {
        Logger.error(err)()
        reject(err)
      })
    })
  },
```

This line:

```js
if (context) context.output.cacheTags.add(`home`);
```

is in charge of assigning the specific tag with current HTTP request output.
