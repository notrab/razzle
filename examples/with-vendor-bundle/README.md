# Razzle with Vendor Bundling

## How to use
Download the example [or clone the whole project](https://github.com/jaredpalmer/razzle.git):

```bash
curl https://codeload.github.com/jaredpalmer/razzle/tar.gz/master | tar -xz --strip=2 razzle-master/examples/with-custom-webpack-config
cd with-custom-webpack-config
```

Install it and run:

```bash
yarn install
yarn start
```

## Idea behind the example
This example demonstrates how to use a `razzle.config.js` file to modify Razzle's
underlying webpack configuration to add a vendor bundle using [Twitter Lite's approach](https://medium.com/@paularmstrong/twitter-lite-and-high-performance-react-progressive-web-apps-at-scale-d28a00e780a3).


First create a file called `razzle.config.js` in your root directory. I won't go into details about the plugins, because they are well document in this [medium article by Paul Armstrong](https://medium.com/@paularmstrong/twitter-lite-and-high-performance-react-progressive-web-apps-at-scale-d28a00e780a3), but read through the comments in the snippet to get your bearings.


```js
'use strict';

module.exports = {
  modify(defaultConfig, { target, dev }, webpack) {
    const config = defaultConfig; // stay immutable here

    // Change the name of the server output file in production
    if (target === 'web') {
      // modify filenaming to account for multiple entry files
      config.output.filename = dev
        ? 'static/js/[name].js'
        : 'static/js/[name].[hash:8].js';

      // add another entry point called vendor
      config.entry.vendor = [
        require.resolve('react'),
        require.resolve('react-dom'),
        // ... add any other vendor packages with require.resolve('xxx')
      ];

      // Use Twitter Lite's vendor & manifest bundle approach
      // See https://medium.com/@paularmstrong/twitter-lite-and-high-performance-react-progressive-web-apps-at-scale-d28a00e780a3
      config.plugins.push(
        new webpack.optimize.CommonsChunkPlugin({
          names: ['vendor', 'manifest'],
          minChunks: Infinity,
        })
      );

      // Extract common modules from all the chunks (requires no 'name' property)
      config.plugins.push(
        new webpack.optimize.CommonsChunkPlugin({
          async: true,
          children: true,
          minChunks: 4,
        })
      );
    }

    return config;
  },
};
```

In `server.js`, we modify our HTML template with our extra js files. 

```jsx
// server.js

// ...


server
  .disable('x-powered-by')
  .use(express.static(process.env.RAZZLE_PUBLIC_DIR))
  .get('/*', (req, res) => {
    const markup = renderToString(<App />);
    res.send(
      `<!doctype html>
    <html lang="">
    <head>
        <meta httpEquiv="X-UA-Compatible" content="IE=edge" />
        <meta charSet='utf-8' />
        <title>Welcome to Razzle</title>
        <meta name="viewport" content="width=device-width, initial-scale=1">
        ${assets.client.css
          ? `<link rel="stylesheet" href="${assets.client.css}">`
          : ''}
        ${process.env.NODE_ENV === 'production'
          ? `<script src="${assets.manifest.js}" defer></script>`
          : `<script src="${assets.manifest.js}" defer crossorigin></script>`}
        ${process.env.NODE_ENV === 'production'
          ? `<script src="${assets.vendor.js}" defer></script>`
          : `<script src="${assets.vendor.js}" defer crossorigin></script>`}
        ${process.env.NODE_ENV === 'production'
          ? `<script src="${assets.client.js}" defer></script>`
          : `<script src="${assets.client.js}" defer crossorigin></script>`}
    </head>
    <body>
        <div id="root">${markup}</div>
    </body>
</html>`
    );
  });
```

Almost done. For the best DX experience, we need to tell Razzle where our development bundle of react is
now located so it can pipe runtime errors into a fancy error overlay instead of to the console.

Create a `.env` file with the relative path to our vendor bundle during development:

```
REACT_DEV_BUNDLE_PATH=/static/js/vendor.js
```

That's it.

```
yarn start
```

Congrats! Your website bundles its JS like m.twitter.com does. In fact, since m.twitter.com doesn't server render...you might even say this setup your build tooling is more advanced!

