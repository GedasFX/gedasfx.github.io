---
layout: post
title: How does Next.js 10 handle imports?
date: 2021-01-31
categories: nextjs performance
---

This is an older writeup I did, might as well leave all my blogs in one place.

**See it on Medium: [How does Next.js 10 handle imports?](https://gedasfx.medium.com/how-does-next-js-10-handle-imports-c51b9e4451c6)**


This page was created 2024-09-15. See original below:


# How does Next.js 10 handle imports?

I was trying to reduce the bundle size of the final client bundle, and I found an interesting feature about Next.js. In one file it handles both server and client side code. This is very convenient, however what happens when you import a third party module at the top of the file? Does the client bundle size get increased if the imported module is only used in `getStaticProps` or `getServerSideProps`?

For a quick answer, see [Conclusions](#conclusions).

## Different kinds of imports

There are many ways to import modules and other files in JavaScript, however all methods can be categorized into two main ways: at importing inline or at the top of the file. An example can be seen here:

```ts
// At the top of the file
import * as fs from 'fs';

export const func = async () => {
  const file = fs.readFileSync('./file');
};

// Inline
export const func = async () => {
  const fs = await import('fs');
  const file = fs.readFileSync('./file');
};
```

If the bundler is not smart about it, the second may provide an advantage.

## Testing

To analyze bundle sizes [@next/bundle-analyzer](https://www.npmjs.com/package/@next/bundle-analyzer) plugin will be used, which is [included in my personal starting template](https://github.com/GedasFX/nextjs-starter). After the build, the plugin would then produce two analysis files (one for server and one for client), which could then be inspected manually. 

For the data set, I will test with 3 common use cases - importing a .json file, reading a file using an import from Node.js ([fs](https://nodejs.org/api/fs.html)), and lastly I will try importing an 3rd party library ([Apollo Client](https://www.npmjs.com/package/@apollo/client)).

### Obtaining data

I went to a [GraphQL API](https://api.graphql.jobs/), and got myself a bit of sample data using the following schema:

```gql
{
  jobs {
    title
    description
    commitment {
      title
    }
    cities {
      name
      country {
        isoCode
        name
      }
    }
  }
}

```

The data ended up being roughly 254KB in size. If there were any optimizations the compilers performed, a change of even 0.01KB would be noticeable in the final bundle.

### Control group

To analyze how much an import adds, lets first check what is the base imports of the project. Again, I use my starter template for this.

```tsx
import { useMemo } from 'react';

export default function Home() {
  const dataString = useMemo(() => {
    return JSON.stringify({});
  }, []);

  return dataString;
}
```

Server: 61.46 KB
![image-20210130193518952](C:\Users\gedim\Desktop\How does Next.js handle imports.assets\image-20210130193518952.png)

Client: 297.87 KB
![image-20210130193449073](C:\Users\gedim\Desktop\How does Next.js handle imports.assets\image-20210130193449073.png)

I tested the size again by adding `getStaticProps` and `getServerSideProps` and, while bundle size increased, the increase was negligible. 

##### Note on Typescript imports

In my files you will see something along the lines of:

```tsx
type Props = {
  jobs: typeof import('data/jobs.json');
};
```

Type declaration DO NOT carry over when the code gets converted from JS from TS, so the bundle size is not going to be affected by it.

## Collecting results

All of my tests and diffs can be found [here](https://github.com/GedasFX/nextjs-import-cost-tests/commits/main). Starting with [this commit](https://github.com/GedasFX/nextjs-import-cost-tests/commit/498f11abc50b2ba74ba462fa99539d054391cfa8) for Static - Nothing, and ending with [SSG - Apollo  (Server + Client)](https://github.com/GedasFX/nextjs-import-cost-tests/commit/6c6b56430e3f6babaa6b3d6790bf8145b6a14ea9). To get the results for yourself, checkout the appropriate commit, and then run

```
$ rm -rf ./.next && yarn run build:analyze && yarn run start
```

This will cleanup .next directory, build and analyze the bundle and start the server on port 3000. 

* **Size** & **First Load JS** stats come from the build output after running `next build`.
* **BA: Client** & **BA: Server** stats come from the generated `./.next/analyze/*.html`.
* **Page** & **Bundle** both transferred and resource come from Google Chrome Dev Tools - Network, where I check Page for the document request, and Bundle for all of the requests.
* **index.*** files come from the generated files - `./.next/server/pages/index.*`.

## Findings

So what are the results? Here they are:

|                                                              | Size | First Load JS | BA: Client | BA: Server | Page: Transferred | Page: Resource | Bundle: Transferred | Bundle: Resource | index .html | index .js | index .json |
| ------------------------------------------------------------ | ---- | ------------- | ---------- | ---------- | ----------------- | -------------- | ------------------- | ---------------- | ----------- | --------- | ----------- |
| Static - Nothing                                             | 0,3  | 66,9          | 297,87     | 61,46      | 1,2               | 2,7            | 72,4                | 206              | 3           | N/A       | N/A         |
|                                                              |      |               |            |            |                   |                |                     |                  |             |           |             |
| [Static - Client](https://github.com/GedasFX/nextjs-import-cost-tests/commit/ad997d5232858a634adcd023d4236f723d8566fd) | 66,7 | 133           | 541,65     | 306,36     | 68,8              | 273            | 207                 | 733              | 267         | N/A       | N/A         |
| [SSG - Server](https://github.com/GedasFX/nextjs-import-cost-tests/commit/f46b03814f26a2235cd2694ca18187726908b0f8) | 0,3  | 66,9          | 297,93     | 306,59     | 134               | 525            | 205                 | 729              | 514         | 258       | 247         |
| [SSR - Server](https://github.com/GedasFX/nextjs-import-cost-tests/commit/9241f4e8765cb74cc1ab74a3d71cc59a98a3118c) | 0,3  | 66,9          | 297,93     | 306,6      | 134               | 525            | 205                 | 729              | N/A         | 258       | N/A         |
| [SSG - Server +  Client](https://github.com/GedasFX/nextjs-import-cost-tests/commit/8e1e2838e87f9dac83595a7acb0f3a681f32d0db) | 66,7 | 133           | 541,75     | 306,7      | 134               | 525            | 272                 | 986              | 514         | 258       | 247         |
|                                                              |      |               |            |            |                   |                |                     |                  |             |           |             |
| [SSG - FS](https://github.com/GedasFX/nextjs-import-cost-tests/commit/0cb0f097b0b5fcab0aeb160a6387008b9ccb2df4) | 0,3  | 66,9          | 297,93     | 62,11      | 134               | 525            | 205                 | 729              | 514         | 6         | 247         |
| [SSG - FS (Dynamic)](https://github.com/GedasFX/nextjs-import-cost-tests/commit/7da4ab2f87a4ff2723bf55c6a86d2186f7fb97a3) | 0,3  | 66,9          | 297,93     | 62,36      | 134               | 525            | 205                 | 729              | 514         | 6         | 247         |
| [SSR - FS (Dynamic)](https://github.com/GedasFX/nextjs-import-cost-tests/commit/ffb5c8b37bdd6bf2d9da56e38866ae914699793c) | 0,3  | 66,9          | 297,93     | 62,37      | 134               | 525            | 205                 | 729              | N/A         | 6         | N/A         |
| [SSR - FS](https://github.com/GedasFX/nextjs-import-cost-tests/commit/a987ccbc4c957cbc5d4d07c5b3aff1e908272d77) | 0,3  | 66,9          | 297,93     | 62,12      | 134               | 525            | 205                 | 729              | N/A         | 6         | N/A         |
|                                                              |      |               |            |            |                   |                |                     |                  |             |           |             |
| [SSG - Apollo  (Server)](https://github.com/GedasFX/nextjs-import-cost-tests/commit/dcfd3d537aa0d9949ac6fc6b74dee890c9035c07) | 0,3  | 66,9          | 297,93     | 62,73      | 134               | 526            | 206                 | 729              | 514         | 6         | 247         |
| [SSG - Apollo  (Server) (Dynamic)](https://github.com/GedasFX/nextjs-import-cost-tests/commit/3a47d314a9074c5290e5106b2acb264abeba8e28) | 0,3  | 66,9          | 297,93     | 62,86      | 134               | 526            | 206                 | 729              | 514         | 6         | 247         |
| [SSR - Apollo  (Server)](https://github.com/GedasFX/nextjs-import-cost-tests/commit/c8cf1f968363fcb7415a35be25437edca9fdcde2) | 0,3  | 66,9          | 297,93     | 62,74      | 134               | 526            | 206                 | 729              | N/A         | 6         | N/A         |
| [SSG - Apollo  (Server + Client)](https://github.com/GedasFX/nextjs-import-cost-tests/commit/6c6b56430e3f6babaa6b3d6790bf8145b6a14ea9) | 45,7 | 112           | 464,15     | 64,08      | 134               | 526            | 251                 | 900              | 514         | 8         | 247         |

Looks like an explanation is necessary. Lets begin with the easiest to understand. The second group (the FS import one), is pretty static. It does make sense, because there is little difference for the client between SSR and SSG. Another finding is the fact that dynamic imports are not necessary. 

### Making sense of the data

#### Rendering: SSG vs SSR

[Next.js docs suggest using SSG over SSR whenever possible](https://nextjs.org/docs/basic-features/data-fetching). This does make sense intuitively, as fetching data and caching it, in 99,9% of the cases would be faster than recreating it every request. Still, I wanted to check what are the compilation differences from SSG to SSR.

To begin from the obvious, as mentioned before, SSR does not generate `.html` or `.json` files, as they would be recreated every request anyway. [This JSON file will be used in client-side routing through `next/link` ](https://nextjs.org/docs/basic-features/data-fetching#statically-generates-both-html-and-json), and the HTML file will be used when initializing first load. With that out of the way, what are the differences in the `.js` file?

```diff
- /* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, "getStaticProps", function() { return getStaticProps; });
+ /* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, "getServerSideProps", function() { return getServerSideProps; });

- const getStaticProps = async () => {
+ const getServerSideProps = async () => {
```

Well it appears, that from code side, nothing unexpected has changed. All that means is that, as the official docs state, **use SSG over SSR whenever possible**.

#### Imports: Static vs Dynamic

For simplicity, lets analyze 2 very similar results. `SSG - FS` and `SSG - FS (Dynamic)` have very similar results. The same translates well when using a third party library. Analyzing the compiled code might give us insight as to why that is. Running `code --diff index1.js index2.js` lets us see how the final JavaScript code was transformed. There are a couple of things to look at but most important is the removal of a harmony import:

```diff
- /* harmony import */ var fs__WEBPACK_IMPORTED_MODULE_1__ = __webpack_require__("mw/K");
```

`"mw/K"` in our case is:

```js
/***/ "mw/K":
/***/ (function(module, exports) {

module.exports = require("fs");

/***/ })
```

So if the module is not imported, where did it go? Well, unsurprisingly, in the module import itself.

```diff
~ const getStaticProps = async () => {
+ const fs = await Promise.resolve(/* import() */).then(__webpack_require__.t.bind(null, "mw/K", 7));
```

The require mode in our case is 0111 (7), which is not important to understand how it works, but considering the last bit is 1, it will work no slower than a regular import:

```js
if(mode & 1) value = __webpack_require__(value);
```

##### Ok what about the client side?

Empirical data suggests that even after we imported a [sizable module](https://bundlephobia.com/result?p=@apollo/client@3.3.7) to be used in server-side, the bundle, which was sent to the client, size didn't change at all. It doesn't matter how they did it (likely a fancier tree-shake), but **modules imported to ONLY be used in `getStaticProps` or `getServerSideProps` do not get bundled into the client-side code**.

With those findings it is safe to say that **unless the method is not going to be called, using dynamic import is actually slower overall**. Considering our use case, `getStaticProps` or `getServerSideProps` will be called regardless, **it is unwise/unnecessary to use dynamic imports**.

#### Bonus: Server-Side modules.

Code from node_modules appears to not get bundled together in the server-side code. This means that the **server will cease to work properly if node_modules folder were to be deleted**.

Consider the following workflow: you have all of the development and client side dependencies described in `"devDependencies"` section of `package.json` file. Then you proceed to build the application with commands `yarn install && yarn build`. Now all of the client side files have been built. To conserve space you delete node_modules, and then install only the packages required for SSR (React, Next, and other 3rd party libraries such as a SQL Client) using `yarn install --prod`. 

To test if this works, `react-icons` package was was installed in `devDependencies`, then a Docker container was built using the following Dockerfile:

```dockerfile
FROM node:lts-alpine AS node_modules
WORKDIR /build

# Install ONLY the packages needed for SSR.
COPY package.json .
COPY yarn.lock .
RUN [ "yarn", "install", "--prod" ]


FROM node:lts-alpine AS build
WORKDIR /build

# Install all of the packages needed for the build.
COPY package.json .
COPY yarn.lock .

# Copy the node_modules needed for production to speed up the installation.
COPY --from=node_modules /build/node_modules ./node_modules
RUN [ "yarn", "install" ]

# Copy the source files and build the program.
COPY . .
RUN [ "yarn", "build" ]


FROM node:lts-alpine AS prod
WORKDIR /app

# Enable production optimizations
ENV NODE_ENV=production

# Copy the files required for rendering.
COPY package.json .
COPY --from=build /build/.next ./.next
COPY --from=build /build/public ./public
COPY --from=node_modules /build/node_modules ./node_modules

EXPOSE 3000
ENTRYPOINT [ "yarn", "start" ]
```

With this, we will only have access to the built files, public folder and non-dev dependencies.

Unfortunately, even though we have this page pre-rendered on build, Next.js decides to render it again, causing an error:

```
$ next start
ready - started server on http://localhost:3000
Error: Cannot find module 'react-icons/fa'
Require stack:
- /app/.next/server/pages/index.js
```

It shows that the error occurred on the `index.js` file. That file is only generated on SSR or SSG. What about Static generation? Well as it turns out, the application does in fact compile without errors. This means **a module can be removed from node_modules if the page was rendered using the Static method (not SSG)**.

There is a workaround though for SSG though: I have accidently discovered that if you use `next/dynamic`, you can specify [to disable SSR](https://nextjs.org/docs/advanced-features/dynamic-import#with-no-ssr), and the module would be precompiled. Another way would be to [export](https://nextjs.org/docs/advanced-features/static-html-export) the files entirely. This however would disable SSR capabilities all together.

All in all, even though client side libraries are not being used on `getStaticProps`, they are still being referenced in the `.js` file, which will stop execution if the library no longer exists in node_modules. **Apart from a few edge cases, to avoid unexpected issues, all client-side libraries must stay in node_modules**.

#### What if a module is being used both in server and client side?

Unsurprisingly, if you reference the client in your client-side component, it gets bundled in by `webpack` and increases the client bundle size. Server "bundle" size does not change, as no bundling is performed on the server side.

## Conclusions

**All imports, which are then used ONLY in either `getStaticProps` or `getServerSideProps` will not be sent over to the client. It is perfectly fine (and likely better) to write the import statement at the top of the file, rather than in the method itself.** If the import is used anywhere else, it will be sent over to the client.

Compiled code wise, the only difference between SSR and SSG, is that SSG generates `.html` and `.json` files which are then read to handle the request, rather than rerunning the method mid request.

There is little difference between static and dynamic imports in the final compiled code. From the looks of things, it is fine to use whatever method, however it appears that the static imports are ever so slightly better.

Do not remove node_modules folder after building the application, because server-side pages do not create a bundle, but rather use references to node_modules. There are a couple of caveats to this, such as if the page is Static (not SSG or SSR), it is fine to remove all node_modules used in that file from node_modules in the final build (assuming they are not used somewhere else).

It is possible to import a file to both server and client side. One reasonable module to import would be the Apollo Client. If you were to import a `.json` file, which would then be used in both server and client, it would be more beneficial to just use it in server side and transfer the data needed as props.

