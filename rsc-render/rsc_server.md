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

这里便是生成`RSC Payload`并在`htmlString`中注入的地方。

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

1. `reactServerStream`是什么？
2. `RSC Payload`是如何注入到`HTML`中的？
  
## reactServerStream
...


