# SpringBoot For GraphQL配置文件参数 <!-- {docsify-ignore-all} -->


## Spring Boot 配置参数（或者application.yml 或application.properties）

```properties
# 设置是否应创建和公开GraphQL servlet。如果未指定，则默认为“true”。
graphql.servlet.enabled=true
# 设置GraphQL servlet将公开的路径。如果未指定，则默认为“/grapql”
graphql.servlet.mapping=/graphql
# 跨域配置
graphql.servlet.cors-enabled=true
graphql.servlet.cors.allow-credentials=true
graphql.servlet.cors.allowed-headers=true
graphql.servlet.cors.allowed-methods=true
graphql.servlet.cors.allowed-origin-patterns=https://***.***.com
graphql.servlet.cors.allowed-origins=https://***.***.com
graphql.servlet.cors.exposed-headers=GET, HEAD, POST
graphql.servlet.cors.max-age=1
# GraphQL异步相关配置
graphql.servlet.async.enabled=true
graphql.servlet.async.threads.max=10
graphql.servlet.async.threads.min=10
graphql.servlet.async.threads.name-prefix=graphql
graphql.servlet.async.timeout=1000
```

## 开发工具GraphiQL相关配置参数（按使用需求配置）

&nbsp; &nbsp; 熟悉`RESTful`API的你可能会知道`Postman`和`Insomnia`之类的工具，因为它们不仅可以帮助我们快速可视化`API`开发，还可以帮助我们更快地完成工作。同样，你可以将`GraphiQL`视为`Postman`或 `Insomnia`。 因为`GraphiQL`是`GraphQL`集成开发环境`(IDE)`。它这是一个强大的工具，可以帮助你直观地构建`GraphQL`查询的工具。

&nbsp; &nbsp; 如果graphql.GraphiQL.enabled属性为true，则可以在root/GraphiQL上访问GraphiQL。注意，GraphQL服务器必须在/grapql/*上下文中可用，才能被GraphiQL发现。

可用的Spring Boot配置参数（application.yml或application.properties）

```yml
graphql:
  graphiql:
    # graphiql的路径
    mapping: /graphiql
    endpoint:
      graphql: /graphql
      subscriptions: /subscriptions
    subscriptions:
      timeout: 30
      reconnect: false
    basePath: /
    enabled: true
    pageTitle: GraphiQL
    cdn:
      enabled: false
      version: latest
    props:
      resources:
        query: query.graphql
        defaultQuery: defaultQuery.graphql
        variables: variables.json
      variables:
        editorTheme: "solarized light"
    headers:
      Authorization: "Bearer <your-token>"
```

## 开发工具Altair相关配置参数（按使用需求配置）

可用的Spring Boot配置参数（application.yml或application.properties）

```yml
graphql:
  altair:
    enabled: true
    # altair的路径
    mapping: /altair
    subscriptions:
      timeout: 30
      reconnect: false
    static:
      base-path: /
    page-title: Altair
    cdn:
      enabled: false
      version: 4.0.2
    options:
      endpoint-url: /graphql
      subscriptions-endpoint: /subscriptions
      initial-settings:
        theme: dracula
      initial-headers:
        Authorization: "Bearer <your-token>"
    resources:
      initial-query: defaultQuery.graphql
      initial-variables: variables.graphql
      initial-pre-request-script: pre-request.graphql
      initial-post-request-script: post-request.graphql
```

## 开发工具Playground相关配置参数（按使用需求配置）

```yml
graphql:
  playground:
    mapping: /playground
    endpoint: /graphql
    subscriptionEndpoint: /subscriptions
    staticPath.base: my-playground-resources-folder
    enabled: true
    pageTitle: Playground
    cdn:
      enabled: false
      version: latest
    settings:
      editor.cursorShape: line
      editor.fontFamily: "'Source Code Pro', 'Consolas', 'Inconsolata', 'Droid Sans Mono', 'Monaco', monospace"
      editor.fontSize: 14
      editor.reuseHeaders: true
      editor.theme: dark
      general.betaUpdates: false
      prettier.printWidth: 80
      prettier.tabWidth: 2
      prettier.useTabs: false
      request.credentials: omit
      schema.polling.enable: true
      schema.polling.endpointFilter: "*localhost*"
      schema.polling.interval: 2000
      schema.disableComments: true
      tracing.hideTracingResponse: true
    headers:
      headerFor: AllTabs
    tabs:
      - name: Example Tab
        query: classpath:exampleQuery.graphql
        headers:
          SomeHeader: Some value
        variables: classpath:variables.json
        responses:
          - classpath:exampleResponse1.json
          - classpath:exampleResponse2.json
```