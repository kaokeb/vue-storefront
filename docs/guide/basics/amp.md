# Google Accelerated Mobile Pages

Google Accelerated Mobile pages are available with Vue Storefront 1.6. By default, the Product and Category pages do support AMP. To get the AMP page, you should use the following URL:
`http://localhost:8000/c/hoodies-and-sweatshirts-24` -> `http://localhost:8000/amp/c/hoodies-and-sweatshirts-24`.
The discovery links are added as `<link rel="amphtml" ...`. The landing pages then distribute the traffic to canonical Vue Storefront links to enable all the features to work (by definition, AMP doesn't support any additional JavaScript).

:::warning Important notice
AMP renderer works ONLY IN PRODUCTION MODE as the [Layouts and advanced output operations
](../core-themes/layouts.md) - specifically `index.html` template switching works only in production. In development mode, there can be CSR/SSR issues so don't even test it in development mode :-)
:::

AMPHTML templates are provided by the `src/themes/default-amp` special purpose theme, which inherits some parts from the `default` theme.
