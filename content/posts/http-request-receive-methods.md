---
date: '2026-06-05T18:21:56+08:00'
draft: false
title: 'RESTFUL请求的前后端传递参数对照'
tags: ["http","vue","java","RESTFul"]
categories: ["big-event"]
---

整理http协议中，axios请求携带参数和对应Java后端Controller中接收方法

# 通用格式
```
axios.method(url, data, config)
//          ↑     ↑     ↑
//        地址  请求体  配置(params/headers等)
```

---
# GET — 查询参数

```js
// axios
request.get('/user', { params: { id: 1, name: '张三' } })

// 实际请求: GET /user?id=1&name=张三
```

```java
// Java
@GetMapping("/user")
public Result get(@RequestParam Integer id, @RequestParam String name) { }
```

---
# POST — JSON 请求体

```js
// axios
request.post('/user', { name: '张三', age: 20 })

// 请求体: {"name":"张三","age":20}
// Content-Type: application/json
```

```java
// Java
@PostMapping("/user")
public Result add(@RequestBody User user) { }
```

---
# POST — 表单（URLSearchParams）

```js
// axios
const params = new URLSearchParams()
params.append('username', 'admin')
params.append('password', '123')
request.post('/login', params)

// 请求体: username=admin&password=123

// Content-Type: application/x-www-form-urlencoded
```

```java
// Java
@PostMapping("/login")
public Result login(@RequestParam String username, @RequestParam String password) { }
```

---
# POST — 文件上传（FormData）

```js
// axios
const formData = new FormData()
formData.append('file', fileObject)
formData.append('description', '头像')
request.post('/upload', formData)
// Content-Type: multipart/form-data
```

```java
// Java
@PostMapping("/upload")
public Result upload(@RequestParam MultipartFile file, @RequestParam String description) { }
```

---
# PUT — JSON 请求体（全量更新）

```js
// axios
request.put('/user', { id: 1, name: '张三', age: 20 })
// 请求体: {"id":1,"name":"张三","age":20}
```

```java
// Java
@PutMapping("/user")
public Result update(@RequestBody User user) { }
```

---
# PATCH — 查询参数（部分更新）

```js
request.patch('/user/avatar', null, { params: { avatarPath: '/img/a.png' } })
// 实际请求: PATCH /user/avatar?avatarPath=/img/a.png
```

```java
// Java
@PatchMapping("/user/avatar")
public Result updateAvatar(@RequestParam String avatarPath) { }
```

---
# PATCH — JSON 请求体（部分更新）

```js
// axios
request.patch('/user', { name: '新名字' })
// 请求体: {"name":"新名字"}
```

```java
// Java
@PatchMapping("/user")
public Result update(@RequestBody User user) { }
```

---
# DELETE — 查询参数

```js
// axios — delete 没有请求体，config 在第二个参数
request.delete('/category', { params: { id: 5 } })
// 实际请求: DELETE /category?id=5
```

```java
// Java
@DeleteMapping("/category")
public Result delete(@RequestParam Integer id) { }
```

---
  速查表

![image-20260605182821932](image-20260605182821932.png)

  核心记忆：
  - 有请求体（POST/PUT/PATCH）：第二个参数放数据，查询参数放第三个参数的 params 里
  - 无请求体（GET/DELETE）：查询参数放第二个参数的 params 里
