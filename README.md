graphql-expand
--

## 概述

基于纯 graphql 的后端服务，提供的 Open API 也是一个 graphql 接口。
对于不熟悉的人来说，可读性不高，使用不便，与现有广泛存在的 restful API 差异明显。

目前发现，将 restful API 转发到 graphql API，是一种比较折中的方案。

针对通用 restful 接口，其中 requestBody 是可选，每个资源有以下七个独立接口，分四种HTTP方法类型：

实际 URI 只有两种格式

1. GET /rest/:resource/:resource_id
2. GET /rest/:resources
3. PUT /rest/:resource/:resource_id             (requestBody)
4. PUT /rest/:resources             (requestBody 有id)
5. POST /rest/:resource             (requestBody)
6. DELETE /rest/:resource/:resource_id 
7. DELETE /rest/:resources          (requestBody)

- resource => graphql field operationName
- requestBody => graphql field variables
- resource_id => graphql fetcher param 

查询单个、更新单个、删除单个必须有 resource_id 参数，其他批量操作使用 requestBody。

两者比较：

将 restful 根据 URI 转化成 graphql

**优点**
1. 性能损失小，只增加一些 if else 来判断，但可能多一次 requestBody 解析
2. 对原 graphql 代码无侵入性
3. 对于公开接口而言，可读性高（针对不熟悉 graphql 的人）
4. 使得对资源的缓存容易在外部实现

**缺点**
1. 丢失类型安全和错误校验功能，错误最终只能在 graphql 处理时才能判断，不能在 restful 转化到 graphql 时准确判断
2. restful 处理转化到 graphql 时，丢失 graphql 的可选择性获取字段，组装等功能
3. RequestBody 无法提前进行校验
4. 需要一个新的转发服务
5. 由于 graphql 必须指定返回字段，所以使用 restful 转换时，只能返回所有字段，也就是不具备选择功能

**难点**
* query 拼接，可以使用自动生成工具，减小难度
* restful 映射到 graphql query

## query generator

利用前端代码生成 graphql query 语句，每个 gql 对应服务端的一个 data fetcher 

- 使用 https://github.com/timqian/gql-generator 前端工具，自动生成 gql。
- 执行 gqlg --schemaFilePath src/main/resources/all.graphql --destDirPath src/main/resources/gql --depthLimit 5

在 all.graphql 中是所有需要转发的 graphql schmea。最终会在 gql/ 目录下生成所有 *.gql 语句。

> 这里自动生成的语句实际会有很多的多余字段，需要排除掉。

## 示例

1. 所有 graphql schema 放在 all.graphql 中
2. 启动 ForwardServer.scala
3. 使用 restful 请求 graphql

**使用 restful 完成 crud**

- GET localhost:8080/rest/userVariables 
    - 将会使用 HTTP 调用 graphql `userVariables: [UserVariable]`
- POST localhost:8080/rest/userVariable 
    - 将会使用 HTTP 调用 graphql `createUserVariable(userVariable: VariableInput!): UserVariable!`
```json
{
    "userVariable": {
        "name": "测试graphql",
        "key": "test_132",
        "valueType": "int",
        "description": "132"
    }
}
```
- DELETE localhost:8080/rest/userVariable/y9pmLdQm # y9pmLdQm是一个HashId
    - 将会使用 HTTP 调用 graphql `deleteUserVariable(id: HashId!): Boolean!`
- PUT localhost:8080/rest/userVariable/y9pmLdQm
    - 将会使用 HTTP 调用 graphql `updateUserVariable(id: HashId!, userVariable: VariableInput!): UserVariable!`
```json
{
    "userVariable": {
        "name": "测试graphql",
        "key": "test_132",
        "valueType": "int",
        "description": "132"
    }
}
```

更新和创建涉及json转义，存在问题，目前只支持，标准 result api 的 crud

1. 查询一个
2. 查询所有
3. 更新
4. 创建
5. 删除
6. 批量删除
7. 批量更新

批量操作，未测