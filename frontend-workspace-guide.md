# 前端项目 Workspace 详尽指南

## 目录

1. [项目结构设计](#项目结构设计)
2. [开发环境配置](#开发环境配置)
3. [包管理与依赖](#包管理与依赖)
4. [构建工具配置](#构建工具配置)
5. [代码规范与质量](#代码规范与质量)
6. [开发工具集成](#开发工具集成)
7. [版本控制策略](#版本控制策略)
8. [测试环境搭建](#测试环境搭建)
9. [部署与CI/CD](#部署与cicd)
10. [性能优化配置](#性能优化配置)
11. [安全性配置](#安全性配置)
12. [团队协作规范](#团队协作规范)

---

## 项目结构设计

### 标准目录结构

```
frontend-project/
├── .vscode/                    # VS Code 配置
│   ├── settings.json
│   ├── extensions.json
│   └── launch.json
├── .github/                    # GitHub 配置
│   ├── workflows/              # GitHub Actions
│   ├── ISSUE_TEMPLATE/
│   └── PULL_REQUEST_TEMPLATE.md
├── public/                     # 静态资源
│   ├── index.html
│   ├── favicon.ico
│   └── manifest.json
├── src/                        # 源代码
│   ├── components/             # 组件
│   │   ├── common/             # 通用组件
│   │   ├── layout/             # 布局组件
│   │   └── features/           # 功能组件
│   ├── pages/                  # 页面组件
│   ├── hooks/                  # 自定义 Hooks
│   ├── utils/                  # 工具函数
│   ├── services/               # API 服务
│   ├── store/                  # 状态管理
│   ├── styles/                 # 样式文件
│   ├── types/                  # TypeScript 类型定义
│   ├── constants/              # 常量定义
│   ├── assets/                 # 静态资源
│   └── App.tsx
├── tests/                      # 测试文件
│   ├── __mocks__/
│   ├── utils/
│   └── setup.ts
├── docs/                       # 项目文档
├── scripts/                    # 构建脚本
├── .env.example               # 环境变量示例
├── .gitignore
├── .eslintrc.js
├── .prettierrc
├── tsconfig.json
├── package.json
├── README.md
└── CHANGELOG.md
```

### 组件组织原则

#### 原子设计模式
```
src/components/
├── atoms/                      # 原子组件
│   ├── Button/
│   ├── Input/
│   └── Icon/
├── molecules/                  # 分子组件
│   ├── SearchBox/
│   ├── FormField/
│   └── Card/
├── organisms/                  # 有机体组件
│   ├── Header/
│   ├── Sidebar/
│   └── ProductList/
└── templates/                  # 模板组件
    ├── PageLayout/
    └── FormLayout/
```

#### 功能模块化组织
```
src/features/
├── auth/
│   ├── components/
│   ├── hooks/
│   ├── services/
│   ├── types/
│   └── index.ts
├── dashboard/
└── profile/
```

---

## 开发环境配置

### Node.js 版本管理

#### 使用 .nvmrc 固定版本
```bash
# .nvmrc
18.17.0
```

#### package.json 引擎配置
```json
{
  "engines": {
    "node": ">=18.17.0",
    "npm": ">=9.0.0"
  }
}
```

### 环境变量管理

#### 环境变量层级
```bash
# .env.local          (本地开发，git ignore)
# .env.development     (开发环境)
# .env.staging         (预发布环境)
# .env.production      (生产环境)
```

#### 示例配置
```bash
# .env.example
REACT_APP_API_URL=https://api.example.com
REACT_APP_APP_NAME=MyApp
REACT_APP_VERSION=$npm_package_version
REACT_APP_BUILD_TIME=

# 数据库配置
DATABASE_URL=
REDIS_URL=

# 第三方服务
SENTRY_DSN=
GOOGLE_ANALYTICS_ID=
```

### 开发服务器配置

#### Vite 配置示例
```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@utils': path.resolve(__dirname, './src/utils'),
      '@types': path.resolve(__dirname, './src/types'),
    }
  },
  server: {
    port: 3000,
    open: true,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, '')
      }
    }
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          utils: ['lodash', 'date-fns']
        }
      }
    }
  }
})
```

---

## 包管理与依赖

### 包管理器选择

#### pnpm (推荐)
```json
{
  "packageManager": "pnpm@8.6.0",
  "scripts": {
    "preinstall": "npx only-allow pnpm"
  }
}
```

#### .npmrc 配置
```bash
# .npmrc
registry=https://registry.npmjs.org/
save-exact=true
auto-install-peers=true
shamefully-hoist=false
```

### 依赖分类管理

#### 核心依赖
```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.8.0",
    "axios": "^1.3.0",
    "zustand": "^4.3.0"
  }
}
```

#### 开发依赖
```json
{
  "devDependencies": {
    "@types/react": "^18.0.0",
    "@types/node": "^18.0.0",
    "@vitejs/plugin-react": "^3.1.0",
    "vite": "^4.1.0",
    "typescript": "^4.9.0",
    "eslint": "^8.0.0",
    "prettier": "^2.8.0",
    "husky": "^8.0.0",
    "lint-staged": "^13.0.0"
  }
}
```

### 依赖安全管理

#### 审计配置
```json
{
  "scripts": {
    "audit": "pnpm audit",
    "audit:fix": "pnpm audit --fix",
    "outdated": "pnpm outdated"
  }
}
```

---

## 构建工具配置

### TypeScript 配置

#### tsconfig.json
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["DOM", "DOM.Iterable", "ES6"],
    "allowJs": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noFallthroughCasesInSwitch": true,
    "module": "ESNext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"],
      "@types/*": ["src/types/*"]
    }
  },
  "include": [
    "src/**/*"
  ],
  "exclude": [
    "node_modules",
    "dist",
    "build"
  ]
}
```

### 构建优化配置

#### 代码分割策略
```typescript
// webpack.config.js 或 vite.config.ts
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // 框架代码
          react: ['react', 'react-dom'],
          // 路由代码
          router: ['react-router-dom'],
          // UI 库
          ui: ['antd', '@mui/material'],
          // 工具库
          utils: ['lodash', 'date-fns', 'axios'],
          // 状态管理
          store: ['zustand', 'redux']
        }
      }
    }
  }
}
```

---

## 代码规范与质量

### ESLint 配置

#### .eslintrc.js
```javascript
module.exports = {
  env: {
    browser: true,
    es2021: true,
    node: true,
  },
  extends: [
    'eslint:recommended',
    '@typescript-eslint/recommended',
    'plugin:react/recommended',
    'plugin:react-hooks/recommended',
    'plugin:jsx-a11y/recommended',
    'plugin:import/recommended',
    'plugin:import/typescript',
    'prettier'
  ],
  parser: '@typescript-eslint/parser',
  parserOptions: {
    ecmaFeatures: {
      jsx: true,
    },
    ecmaVersion: 'latest',
    sourceType: 'module',
  },
  plugins: [
    'react',
    '@typescript-eslint',
    'react-hooks',
    'jsx-a11y',
    'import'
  ],
  rules: {
    'react/react-in-jsx-scope': 'off',
    'react/prop-types': 'off',
    '@typescript-eslint/no-unused-vars': 'error',
    '@typescript-eslint/explicit-function-return-type': 'warn',
    'import/order': [
      'error',
      {
        groups: [
          'builtin',
          'external',
          'internal',
          'parent',
          'sibling',
          'index'
        ],
        'newlines-between': 'always',
        alphabetize: {
          order: 'asc',
          caseInsensitive: true
        }
      }
    ]
  },
  settings: {
    react: {
      version: 'detect',
    },
    'import/resolver': {
      typescript: {
        alwaysTryTypes: true,
      },
    },
  },
}
```

### Prettier 配置

#### .prettierrc
```json
{
  "semi": false,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 80,
  "bracketSpacing": true,
  "arrowParens": "avoid",
  "endOfLine": "lf"
}
```

### Git Hooks 配置

#### husky + lint-staged
```json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
    }
  },
  "lint-staged": {
    "src/**/*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "src/**/*.{css,scss,less}": [
      "stylelint --fix",
      "prettier --write"
    ]
  }
}
```

### 提交信息规范

#### commitlint 配置
```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',     // 新功能
        'fix',      // 修复
        'docs',     // 文档
        'style',    // 格式
        'refactor', // 重构
        'perf',     // 性能
        'test',     // 测试
        'chore',    // 构建
        'revert'    // 回滚
      ]
    ]
  }
}
```

---

## 开发工具集成

### VS Code 配置

#### .vscode/settings.json
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true,
    "source.organizeImports": true
  },
  "typescript.preferences.importModuleSpecifier": "relative",
  "emmet.includeLanguages": {
    "typescript": "html",
    "typescriptreact": "html"
  },
  "files.associations": {
    "*.css": "tailwindcss"
  }
}
```

#### .vscode/extensions.json
```json
{
  "recommendations": [
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint",
    "bradlc.vscode-tailwindcss",
    "ms-vscode.vscode-typescript-next",
    "formulahendry.auto-rename-tag",
    "christian-kohler.path-intellisense",
    "ms-vscode.vscode-json"
  ]
}
```

### 调试配置

#### .vscode/launch.json
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Chrome Debug",
      "type": "chrome",
      "request": "launch",
      "url": "http://localhost:3000",
      "webRoot": "${workspaceFolder}/src",
      "sourceMapPathOverrides": {
        "webpack:///src/*": "${webRoot}/*"
      }
    }
  ]
}
```

---

## 版本控制策略

### Git 配置

#### .gitignore
```bash
# 依赖
node_modules/
.pnp
.pnp.js

# 生产构建
/build
/dist

# 环境变量
.env.local
.env.development.local
.env.test.local
.env.production.local

# 日志
npm-debug.log*
yarn-debug.log*
yarn-error.log*
lerna-debug.log*

# 缓存
.eslintcache
.stylelintcache

# IDE
.vscode/
.idea/
*.swp
*.swo

# 操作系统
.DS_Store
Thumbs.db

# 测试覆盖率
coverage/
.nyc_output

# 调试
.vscode/
```

### 分支策略

#### Git Flow 模式
```bash
# 主分支
main          # 生产版本
develop       # 开发主线

# 功能分支
feature/*     # 新功能开发
release/*     # 发布准备
hotfix/*      # 紧急修复
```

#### 分支命名规范
```bash
feature/user-authentication
feature/payment-integration
bugfix/login-validation
hotfix/security-patch
release/v1.2.0
```

---

## 测试环境搭建

### 测试框架配置

#### Jest + Testing Library
```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --watchAll=false"
  }
}
```

#### jest.config.js
```javascript
module.exports = {
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/src/setupTests.ts'],
  moduleNameMapping: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@components/(.*)$': '<rootDir>/src/components/$1'
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/index.tsx',
    '!src/reportWebVitals.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
}
```

### E2E 测试配置

#### Playwright 配置
```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
  ],
  webServer: {
    command: 'npm run dev',
    port: 3000,
  },
})
```

---

## 部署与CI/CD

### GitHub Actions 配置

#### .github/workflows/ci.yml
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x, 20.x]
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'pnpm'
    
    - name: Install dependencies
      run: pnpm install --frozen-lockfile
    
    - name: Run linting
      run: pnpm run lint
    
    - name: Run type checking
      run: pnpm run type-check
    
    - name: Run tests
      run: pnpm run test:ci
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'pnpm'
    
    - name: Install dependencies
      run: pnpm install --frozen-lockfile
    
    - name: Build application
      run: pnpm run build
    
    - name: Upload build artifacts
      uses: actions/upload-artifact@v3
      with:
        name: build-files
        path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Deploy to production
      run: echo "Deploy to production server"
```

### Docker 配置

#### Dockerfile
```dockerfile
# 多阶段构建
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
COPY pnpm-lock.yaml ./

RUN npm install -g pnpm
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm run build

# 生产环境
FROM nginx:alpine AS production

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### docker-compose.yml
```yaml
version: '3.8'

services:
  frontend:
    build: .
    ports:
      - "3000:80"
    environment:
      - NODE_ENV=production
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - frontend
```

---

## 性能优化配置

### 构建优化

#### Bundle 分析
```json
{
  "scripts": {
    "analyze": "vite-bundle-analyzer dist",
    "build:analyze": "npm run build && npm run analyze"
  }
}
```

#### 代码分割配置
```typescript
// 路由懒加载
const Home = lazy(() => import('@/pages/Home'))
const About = lazy(() => import('@/pages/About'))

// 组件懒加载
const HeavyComponent = lazy(() => import('@/components/HeavyComponent'))
```

### 缓存策略

#### Service Worker 配置
```javascript
// public/sw.js
const CACHE_NAME = 'app-cache-v1'
const urlsToCache = [
  '/',
  '/static/js/bundle.js',
  '/static/css/main.css'
]

self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(urlsToCache))
  )
})
```

### 图片优化

#### 响应式图片配置
```typescript
// 图片组件
interface ImageProps {
  src: string
  alt: string
  sizes?: string
  loading?: 'lazy' | 'eager'
}

const OptimizedImage: React.FC<ImageProps> = ({ 
  src, 
  alt, 
  sizes = '100vw',
  loading = 'lazy' 
}) => {
  return (
    <picture>
      <source 
        srcSet={`${src}?w=480 480w, ${src}?w=800 800w`}
        sizes={sizes}
        type="image/webp"
      />
      <img 
        src={src} 
        alt={alt} 
        loading={loading}
        decoding="async"
      />
    </picture>
  )
}
```

---

## 安全性配置

### 内容安全策略

#### CSP 配置
```html
<!-- index.html -->
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; 
               script-src 'self' 'unsafe-inline'; 
               style-src 'self' 'unsafe-inline'; 
               img-src 'self' data: https:;">
```

### 环境变量安全

#### 敏感信息处理
```typescript
// 环境变量验证
const requiredEnvVars = [
  'REACT_APP_API_URL',
  'REACT_APP_APP_NAME'
] as const

const validateEnv = (): void => {
  for (const envVar of requiredEnvVars) {
    if (!process.env[envVar]) {
      throw new Error(`Missing required environment variable: ${envVar}`)
    }
  }
}

validateEnv()
```

### API 安全配置

#### Axios 拦截器
```typescript
// 请求拦截器
axios.interceptors.request.use(
  config => {
    const token = localStorage.getItem('token')
    if (token) {
      config.headers.Authorization = `Bearer ${token}`
    }
    return config
  },
  error => Promise.reject(error)
)

// 响应拦截器
axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      // 清除认证信息
      localStorage.removeItem('token')
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)
```

---

## 团队协作规范

### 代码审查流程

#### PR 模板
```markdown
## 变更说明
简要描述此次变更的内容

## 变更类型
- [ ] 新功能
- [ ] Bug 修复
- [ ] 重构
- [ ] 文档更新
- [ ] 性能优化

## 测试情况
- [ ] 单元测试通过
- [ ] E2E 测试通过
- [ ] 手动测试完成

## 检查清单
- [ ] 代码符合项目规范
- [ ] 添加了必要的测试
- [ ] 更新了相关文档
- [ ] 考虑了向后兼容性
```

### 文档规范

#### README 模板
```markdown
# 项目名称

## 快速开始

### 环境要求
- Node.js >= 18.17.0
- pnpm >= 8.0.0

### 安装依赖
```bash
pnpm install
```

### 启动开发服务器
```bash
pnpm dev
```

## 项目结构
详细的项目结构说明...

## 开发指南
详细的开发流程说明...

## 部署指南
详细的部署流程说明...
```

### 发布流程

#### 版本管理
```json
{
  "scripts": {
    "version:patch": "npm version patch",
    "version:minor": "npm version minor", 
    "version:major": "npm version major",
    "release": "npm run build && npm run test && npm publish"
  }
}
```

#### 变更日志
```markdown
# Changelog

## [1.2.0] - 2024-01-15

### Added
- 新增用户认证功能
- 添加暗色主题支持

### Changed
- 优化页面加载性能
- 更新 UI 组件库

### Fixed
- 修复登录状态丢失问题
- 解决移动端适配问题

### Removed
- 移除过时的 API 接口
```

---

## 总结

本指南涵盖了前端项目 workspace 的各个重要方面，从项目初始化到生产部署的完整流程。关键要点：

1. **标准化配置**：确保团队成员使用相同的开发环境和工具配置
2. **自动化流程**：通过 CI/CD 和 Git hooks 自动化代码质量检查和部署
3. **性能优化**：从构建配置到运行时优化的全方位性能提升
4. **安全保障**：实施全面的安全策略保护应用和用户数据
5. **团队协作**：建立清晰的协作规范和流程

定期审查和更新这些配置，确保项目始终保持最佳实践和现代化标准。