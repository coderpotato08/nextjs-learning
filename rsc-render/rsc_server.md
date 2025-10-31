# RSC 服务端

## 服务端生成RSC负载

我们都知道`Next.js`会在服务端生成最终`HTML`字符串，并流式的传递给客户端

服务端最终会通过调用`renderToStream`来生成传递给客户端的`HTML`流

```ts
// packages/next/src/server/app-render/app-render.tsx
export const renderToHTMLOrFlight: AppPageRender = (
  req,
  // ... params
) => {
  return workAsyncStorage.run(
    workStore,
    // The function to run
    renderToHTMLOrFlightImpl,
    // all of it's args
    // ... params
  )
}

...

async function renderToHTMLOrFlightImpl(
  req: BaseNextRequest,
  // ... params
) {
  // ...
  const renderToStreamWithTracing = getTracer().wrap(
    AppRenderSpan.getBodyResult,
    {
      spanName: `render route (app) ${pagePath}`,
      attributes: {
        'next.route': pagePath,
      },
    },
    renderToStream // 调用renderToStream 生成htmlString
  )
  // ...
}
```

`renderToStream`中会引用到最终渲染到客户端的`Root`组件`<APP />`

```tsx
const htmlStream = await workUnitAsyncStorage.run(
  requestStore,
  resume,
  <App
    reactServerStream={reactServerResult.tee()}
    reactDebugStream={reactDebugStream}
    preinitScripts={preinitScripts}
    clientReferenceManifest={clientReferenceManifest}
    ServerInsertedHTMLProvider={ServerInsertedHTMLProvider}
    nonce={nonce}
    images={ctx.renderOpts.images}
  />,
  postponed,
  { onError: htmlRendererErrorHandler, nonce }
)
```

- `workUnitAsyncStorage`可以看作是`node`的`AsyncLocalStorage`, 用来统一管理异步状态，后面会统一介绍`AsyncLocalStorage`在`Next.js`中的应用
- `<App />`组件便是最终会渲染到客户端的`Root`组件，`RSC Payload`便是在其内部初始化注入的
  
## useFlightStream

这里便是将`RSC Payload`注入`htmlString`的地方。

```tsx
// packages/next/src/server/app-render/app-render.tsx
const response = React.use(
  useFlightStream<InitialRSCPayload>(
    reactServerStream,
    reactDebugStream,
    clientReferenceManifest,
    nonce
  )
)
```
`useFlightStream`会在`reactServerStream`中收集`RSC Payload`，随后会将其注入到`htmlString`的`__next_f`中


通过上面`useFlightStream`大致流程的介绍，我们可以先设立两个出发点🙋

1. `RSC Payload`是如何生成的？
2. `reactServerStream`是什么？

## RSC Payload

`RSC Payload`是`Next.js`在服务端生成的`RSC`负载，它会被注入到`htmlString`的`__next_f`中，并在客户端通过`__next_f`进行渲染

`RSC Payload`的通过`generateDynamicRSCPayload` 来生成

```ts
/** packages/next/src/server/app-render/app-render.tsx */
const RSCPayload: RSCPayload & {
  /** Only available during cacheComponents development builds. Used for logging errors. */
  _validation?: Promise<React.ReactNode>
  /** asyncLocakStorage.run(store, () => {}, ...args) */
} = await workUnitAsyncStorage.run(
  requestStore,
  generateDynamicRSCPayload,
  ctx,
  options
)
```
最终会生成如下的数据结构

```ts
// server action response
if (options?.actionResult) {
  return {
    a: options.actionResult,
    f: flightData,
    b: ctx.sharedContext.buildId,
  }
}

//  RSC response.
return {
  b: ctx.sharedContext.buildId,
  f: flightData,
  S: workStore.isStaticGeneration,
}
```
flightData 是 RSC 负载的一部分，它包含了 RSC 组件的渲染信息和状态。



