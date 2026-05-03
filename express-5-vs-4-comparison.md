# Express 5 vs Express 4: 完整对比指南

## 概述

Express 5.0 于 2024年10月15日正式发布，距离第一个主版本（2014年）已经过去了10年。这个新版本专注于简化代码库、提高安全性，并放弃对旧版本 Node.js 的支持，以实现更好的性能和可维护性。

## 系统要求

### Express 4
- Node.js 版本 >= 8.0

### Express 5  
- **Node.js 版本 >= 18.0** (重大变更)

```bash
# 安装 Express 5
npm install "express@5"
```

## 主要新功能

### 1. 自动异步错误处理 ⭐

**Express 4:**
```javascript
app.get('/data', async (req, res, next) => {
  try {
    const result = await fetchData();
    res.send(result);
  } catch (err) {
    next(err); // 必须手动处理
  }
});
```

**Express 5:**
```javascript
app.get('/data', async (req, res) => {
  const result = await fetchData(); // 自动错误处理
  res.send(result);
});
```

Express 5 自动将被拒绝的 Promise 转发给错误处理中间件，无需手动 try/catch。

### 2. Brotli 压缩支持

Express 5 现在支持 Brotli 压缩，相比 gzip 提供更好的压缩比：

```javascript
app.use(compression({ level: 6, threshold: 100 * 1000, filter: shouldCompress }));
```

### 3. 增强的安全性和路由匹配

- 升级 `path-to-regexp` 库从 0.x 到 8.x
- 防御正则表达式拒绝服务 (ReDoS) 攻击
- 不再支持子表达式，如 `/:foo(\\d+)`

## 重大变更 (Breaking Changes)

### 1. 方法名称变更

**移除的方法:**
```javascript
// Express 4 ❌
app.del('/user', handler);
req.acceptsCharset();
req.acceptsEncoding(); 
req.acceptsLanguage();

// Express 5 ✅  
app.delete('/user', handler);
req.acceptsCharsets();
req.acceptsEncodings();
req.acceptsLanguages();
```

### 2. 请求参数变更

**Express 4:**
```javascript
// req.param() 方法被移除
app.get('/user/:id', (req, res) => {
  const id = req.param('id'); // ❌ 不再支持
});
```

**Express 5:**
```javascript
app.get('/user/:id', (req, res) => {
  const id = req.params.id || req.body.id || req.query.id; // ✅
});
```

### 3. 响应方法变更

**Express 4:**
```javascript
// ❌ 不再支持
res.json(obj, status);
res.redirect('back');
```

**Express 5:**
```javascript
// ✅ 新语法
res.status(status).json(obj);
res.redirect(req.get('Referrer') || '/');
```

### 4. 路由语法变更

**Express 4:**
```javascript
// 可选参数语法
app.get('/user/:id?', (req, res) => {
  res.send(req.params.id || 'No ID');
});
```

**Express 5:**
```javascript
// 新的可选参数语法
app.get('/user{/:id}', (req, res) => {
  res.send(req.params.id || 'No ID');
});
```

### 5. Body 解析更新

**主要变更:**
- 移除了 `bodyParser()` 中间件
- `req.body` 不再自动初始化为 `{}`
- `urlencoded` 解析器默认 `extended: false`
- 可自定义 URL 编码体的深度限制（默认值：32）

```javascript
// Express 5 推荐用法
app.use(express.json());
app.use(express.urlencoded({ extended: false }));
```

### 6. 状态码验证

Express 5 现在会对无效的状态码抛出错误：

```javascript
// Express 4: 静默失败
res.status(999).send('OK'); 

// Express 5: 抛出错误
res.status(999).send('OK'); // ❌ Error: Invalid status code: 999
```

## 已移除的功能

| 功能 | Express 4 | Express 5 | 替代方案 |
|------|-----------|-----------|----------|
| `app.del()` | ✅ | ❌ | `app.delete()` |
| `req.param()` | ✅ | ❌ | `req.params`, `req.body`, `req.query` |
| `res.sendfile()` | ✅ | ❌ | `res.sendFile()` |
| `res.json(obj, status)` | ✅ | ❌ | `res.status(status).json(obj)` |
| `res.redirect('back')` | ✅ | ❌ | `res.redirect(req.get('Referrer') \|\| '/')` |

## 性能改进

1. **更好的错误处理性能** - 自动异步错误处理减少了样板代码
2. **Brotli 压缩** - 更好的压缩比，减少传输大小
3. **更安全的路由匹配** - 防止 ReDoS 攻击
4. **现代 Node.js 支持** - 利用 Node.js 18+ 的性能优化

## 迁移工具

Express 提供官方 codemods 来自动化迁移：

```bash
# 运行所有 codemods
npx @expressjs/codemod upgrade

# 运行特定的 codemod
npx @expressjs/codemod name-of-the-codemod
```

## 迁移检查清单

### 前期准备
- [ ] 确保 Node.js 版本 >= 18
- [ ] 备份现有项目
- [ ] 运行现有测试套件

### 代码更新
- [ ] 替换 `app.del()` 为 `app.delete()`
- [ ] 更新 `req.acceptsCharset()` 等方法
- [ ] 修复路由可选参数语法
- [ ] 更新响应方法调用
- [ ] 移除 `req.param()` 使用
- [ ] 更新 body 解析中间件

### 测试验证
- [ ] 运行 codemods
- [ ] 执行完整测试套件
- [ ] 验证异步错误处理
- [ ] 检查路由匹配行为
- [ ] 性能基准测试

## 示例：完整迁移

**Express 4 代码:**
```javascript
const express = require('express');
const app = express();

app.use(express.bodyParser());

app.del('/user/:id?', async (req, res, next) => {
  try {
    const id = req.param('id');
    await deleteUser(id);
    res.json({success: true}, 200);
  } catch (err) {
    next(err);
  }
});

app.get('/redirect', (req, res) => {
  res.redirect('back');
});
```

**Express 5 迁移后:**
```javascript
const express = require('express');
const app = express();

app.use(express.json());
app.use(express.urlencoded({ extended: false }));

app.delete('/user{/:id}', async (req, res) => {
  const id = req.params.id;
  await deleteUser(id); // 自动错误处理
  res.status(200).json({success: true});
});

app.get('/redirect', (req, res) => {
  res.redirect(req.get('Referrer') || '/');
});
```

## 总结

Express 5 带来了显著的现代化改进：

**优势:**
- 🚀 自动异步错误处理
- 🔒 增强的安全性
- 📈 更好的性能
- 🛠️ 更清晰的 API

**挑战:**
- 💥 重大变更需要仔细迁移
- 📋 需要 Node.js 18+
- 🔧 可能需要更新依赖项

虽然迁移需要一定的工作量，但 Express 5 提供的现代化功能和性能改进使其成为值得升级的版本。使用官方提供的迁移工具可以显著简化这个过程。

---

*最后更新: 2025年8月*
*Express 版本: 5.0.0*