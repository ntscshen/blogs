---
title: "nestjs 控制器顺序的重要性"
summary: "控制器顺序导致的固定路径被将解释为动态参数"
date: "2024-06-12"
draft: false
tags:
- React
- antd
---

## 控制器顺序

```typescript
@Get(':id')
findOne(@Param('id') id: string) {
  return this.usersService.findOne(+id);
}

@Get('org')
orgList() {
  return this.usersService.orgList();
}
```

在 **postman** 中测试当前地址 **http://localhost:3000/api/org** 命中的却是 `findOne` 这是为什么？

### 如何解决？

1. 方案1：确保**路径参数**的路由定义放在**明确路径**的路由之后。也就是 确保 `@Get('org')` 放在 `@Get(':id')` 之前
2. 方案2：为**路径参数**添加更加明确的前缀，例如 `@Get('user/:id')`

理论上，这些操作不应该在代码层面去做处理， `nestjs` 应该能够区分固定路径和参数路径的差异。

**nestjs** 的路由系统默认依赖于 **Express.js** ，你可以将**Express.js** 改成 **fastify** 但是问题依旧会出现。这是为什么？

### 为什么如此设计路由的匹配规则？

最实际的问题，路由匹配需要解决确定路由的**优先级**并确保路由匹配的**性能**

1. 路由优先级：按照路由声明顺序匹配路由
2. 路由想你能：简单按照声明顺序进行匹配，避免复杂的正则表达式匹配

理解了路由设计的部分原理之后，可能还有有疑惑， `/org`、`/:id` 完全是两个不同的路由，为什么会将 `org`匹配到`:id`中？
这个里面的核心问题就是它们两个路径的最大区别` ":" `

1. 在现代 Web 框架中，如 Express.js、Fastify 和 NestJS，: 被用来定义路径参数。路径参数允许在 URL 中捕获动态值。
2. **路径参数（:）**：用于定义路径中的动态部分，可以捕获特定路径段并将其作为参数传递给处理函数。例如，`/users/:id` 会匹配 `/users/123`，并将 123 作为参数传递给处理函数。
3. **（:）**又叫路径参数通配符：
   1. `/users/:id` 明确表示 **id** 是一个动态值，而不是固定路径的一部分。
   2. 允许捕获特定路径段并将其作为参数传递给处理函数
   3. 使用 : 作为路径参数通配符，可以与 * 等其他通配符区分开来，避免混淆。

既然是路径通配符，那么它就会匹配任何路径段，包括 **/org** ， 通配符路径（如 **/:id**）会匹配任何符合模式的路径，因此如果声明顺序不当，通配符路径可能会优先匹配。

因此在开发过程中，应该牢记这些路由设计的最佳实践：

1. **优先声明具体路径**：始终将具体路径放在动态路径之前，以确保正确的路径匹配。
2. **路径参数后置**：将路径参数放在路由声明的后面，确保具体路径优先匹配。
3. **使用前缀避免冲突**：为路径参数添加明确的前缀，避免与其他路径冲突。
4. **分组路由**：将相关路由分组，并在组内管理匹配顺序。
5. **兜底路由**：使用 * 作为兜底路由，捕获所有未被匹配的请求。

从这些最佳实践回过头看当前碰到的问题，我们只需要将**固定路径地址**放到**动态路径**之前即可

```typescript
@Get(':id')
findOne(@Param('id') id: string) {
  return this.usersService.findOne(+id);
}

@Get('org')
orgList() {
  return this.usersService.orgList();
}
```

## 为什么 Express.js 和 fastify 不在框架层面解决这些问题？

有很多的方案去实现类似功能，例如：

1. 可以内置更智能的路由优先级规则，以自动处理路径冲突
2. 优化路由匹配算法，通过预编译和缓存路径匹配规则，减少匹配时的性能开销
3. 可以在启动时检测路由冲突，并在控制台输出警告或错误信息，提示开发者修改路由定义
4. ...

Express 和 Fastify 等流行框架没有实现这些方案，核心原因是

1. **它们的设计哲学是简单、轻量和高性能**：提供一个最小核心，使开发者可以自由选择如何构建他们的应用。引入复杂的路由匹配逻辑和自动化的优先级管理可能会违背这一设计哲学，增加不必要的复杂性和潜在的性能开销。
2. **给予开发者更多控制权和灵活性**：保持核心的简单性，开发者可以根据他们的需求选择和构建自己的路由管理逻辑。如果框架默认引入复杂的优先级管理和冲突检测机制，会限制开发者的自由度
3. **性能是这些框架的核心优势之一**：添加智能路由匹配和冲突检测机制会增加额外的计算开销，影响路由匹配的速度。在高并发场景下，性能损耗可能会变得显著。保持简单的路由匹配规则可以最大限度地提升性能。
4. **设计哲学的一致性**： **Express** 和 **Fastify** 通过简单而一致的设计赢得了大量用户的青睐。引入复杂的机制会破坏这种一致性，导致开发者在学习和使用过程中面临**更高的学习曲线**和使用障碍。
