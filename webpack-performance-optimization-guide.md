# Webpack 性能优化详尽指南

## 目录

1. [性能分析与测量](#1-性能分析与测量)
2. [代码拆分策略](#2-代码拆分策略)
3. [Tree Shaking 优化](#3-tree-shaking-优化)
4. [构建性能优化](#4-构建性能优化)
5. [运行时性能优化](#5-运行时性能优化)
6. [资源优化](#6-资源优化)
7. [缓存策略](#7-缓存策略)
8. [开发与生产环境优化](#8-开发与生产环境优化)
9. [高级优化技术](#9-高级优化技术)
10. [监控与调试](#10-监控与调试)

---

## 1. 性能分析与测量

### 1.1 Bundle 分析工具

#### webpack-bundle-analyzer
```bash
npm install --save-dev webpack-bundle-analyzer
```

```javascript
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      openAnalyzer: false,
      reportFilename: 'bundle-report.html'
    })
  ]
};
```

#### 其他分析工具
- **webpack-dashboard**: 实时构建信息
- **size-limit**: 包大小限制检查
- **bundlephobia**: 依赖包大小分析

### 1.2 性能指标监控

```javascript
// 构建时间监控
const SpeedMeasurePlugin = require('speed-measure-webpack-plugin');
const smp = new SpeedMeasurePlugin();

module.exports = smp.wrap({
  // webpack 配置
});
```

```javascript
// 运行时性能监控
const { PerformanceObserver, performance } = require('perf_hooks');

const obs = new PerformanceObserver((list) => {
  console.log(list.getEntries());
});
obs.observe({ entryTypes: ['measure'] });
```

---

## 2. 代码拆分策略

### 2.1 入口点拆分 (Entry Points)

```javascript
// webpack.config.js
module.exports = {
  entry: {
    main: './src/index.js',
    vendor: ['react', 'react-dom', 'lodash'],
    admin: './src/admin.js'
  },
  optimization: {
    splitChunks: {
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
      },
    },
  },
};
```

### 2.2 动态导入 (Dynamic Imports)

```javascript
// 路由级别代码拆分
const HomePage = React.lazy(() => import('./components/HomePage'));
const AboutPage = React.lazy(() => import('./components/AboutPage'));

// 条件加载
async function loadFeature() {
  if (shouldLoadFeature) {
    const { feature } = await import('./feature');
    return feature;
  }
}

// 预加载
import(
  /* webpackChunkName: "heavy-feature" */
  /* webpackPrefetch: true */
  './heavy-feature'
);
```

### 2.3 高级拆分配置

```javascript
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      minSize: 20000,
      maxSize: 244000,
      cacheGroups: {
        default: {
          minChunks: 2,
          priority: -20,
          reuseExistingChunk: true
        },
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          priority: -10,
          reuseExistingChunk: true,
          chunks: 'all'
        },
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: 'react',
          priority: 20,
          chunks: 'all'
        },
        antd: {
          test: /[\\/]node_modules[\\/]antd[\\/]/,
          name: 'antd',
          priority: 15,
          chunks: 'all'
        }
      }
    }
  }
};
```

---

## 3. Tree Shaking 优化

### 3.1 基础配置

```javascript
// webpack.config.js
module.exports = {
  mode: 'production',
  optimization: {
    usedExports: true,
    sideEffects: false
  }
};
```

```json
// package.json
{
  "sideEffects": false
}
```

### 3.2 精确的 sideEffects 配置

```json
{
  "sideEffects": [
    "*.css",
    "*.scss",
    "./src/polyfills.js",
    "./src/global-styles.js"
  ]
}
```

### 3.3 ES6 模块优化

```javascript
// 避免导入整个库
// ❌ 不好的做法
import _ from 'lodash';
import { Button } from 'antd';

// ✅ 好的做法
import debounce from 'lodash/debounce';
import Button from 'antd/es/button';
```

### 3.4 Babel 配置优化

```javascript
// babel.config.js
module.exports = {
  presets: [
    ['@babel/preset-env', {
      modules: false, // 保持 ES6 模块
      useBuiltIns: 'usage',
      corejs: 3
    }]
  ],
  plugins: [
    ['import', {
      libraryName: 'antd',
      libraryDirectory: 'es',
      style: true
    }]
  ]
};
```

---

## 4. 构建性能优化

### 4.1 并行处理

```javascript
// thread-loader
module.exports = {
  module: {
    rules: [
      {
        test: /\.js$/,
        use: [
          'thread-loader',
          'babel-loader'
        ]
      }
    ]
  }
};

// parallel-webpack
const parallel = require('parallel-webpack');
module.exports = parallel([
  require('./webpack.config.js'),
  require('./webpack.config.worker.js')
]);
```

### 4.2 缓存策略

```javascript
module.exports = {
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename]
    }
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        use: {
          loader: 'babel-loader',
          options: {
            cacheDirectory: true,
            cacheCompression: false
          }
        }
      }
    ]
  }
};
```

### 4.3 增量构建

```javascript
module.exports = {
  watchOptions: {
    ignored: /node_modules/,
    aggregateTimeout: 300,
    poll: 1000
  },
  optimization: {
    moduleIds: 'deterministic',
    chunkIds: 'deterministic'
  }
};
```

### 4.4 优化 resolve 配置

```javascript
module.exports = {
  resolve: {
    extensions: ['.js', '.jsx', '.ts', '.tsx'],
    alias: {
      '@': path.resolve(__dirname, 'src'),
      'react': path.resolve('./node_modules/react')
    },
    modules: [
      path.resolve(__dirname, 'src'),
      'node_modules'
    ],
    symlinks: false,
    cacheWithContext: false
  }
};
```

---

## 5. 运行时性能优化

### 5.1 懒加载实现

```javascript
// 图片懒加载
const LazyImage = ({ src, alt }) => {
  const [imageSrc, setImageSrc] = useState('');
  const imgRef = useRef();

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setImageSrc(src);
          observer.disconnect();
        }
      },
      { threshold: 0.1 }
    );

    if (imgRef.current) {
      observer.observe(imgRef.current);
    }

    return () => observer.disconnect();
  }, [src]);

  return <img ref={imgRef} src={imageSrc} alt={alt} />;
};

// 组件懒加载
const LazyComponent = React.lazy(() => 
  import('./Component').then(module => ({ 
    default: module.Component 
  }))
);
```

### 5.2 预加载策略

```javascript
// 预加载关键资源
const preloadRoute = (routeName) => {
  const componentImport = routes[routeName];
  componentImport();
};

// 鼠标悬停时预加载
const NavLink = ({ to, children }) => {
  const handleMouseEnter = () => {
    preloadRoute(to);
  };

  return (
    <Link to={to} onMouseEnter={handleMouseEnter}>
      {children}
    </Link>
  );
};
```

### 5.3 Service Worker 缓存

```javascript
// sw.js
const CACHE_NAME = 'app-v1';
const urlsToCache = [
  '/',
  '/static/js/bundle.js',
  '/static/css/main.css'
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then((cache) => cache.addAll(urlsToCache))
  );
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then((response) => {
        return response || fetch(event.request);
      })
  );
});
```

---

## 6. 资源优化

### 6.1 图片优化

```javascript
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.(png|jpe?g|gif|svg)$/i,
        use: [
          {
            loader: 'file-loader',
            options: {
              name: '[path][name].[contenthash].[ext]',
            },
          },
          {
            loader: 'image-webpack-loader',
            options: {
              mozjpeg: {
                progressive: true,
                quality: 65
              },
              optipng: {
                enabled: false,
              },
              pngquant: {
                quality: [0.65, 0.90],
                speed: 4
              },
              gifsicle: {
                interlaced: false,
              },
              webp: {
                quality: 75
              }
            }
          }
        ]
      }
    ]
  }
};
```

### 6.2 字体优化

```css
/* 字体预加载 */
@font-face {
  font-family: 'CustomFont';
  src: url('./fonts/custom.woff2') format('woff2');
  font-display: swap;
}

/* 字体子集化 */
@font-face {
  font-family: 'OptimizedFont';
  src: url('./fonts/optimized-latin.woff2') format('woff2');
  unicode-range: U+0000-00FF, U+0131, U+0152-0153;
}
```

### 6.3 CSS 优化

```javascript
// CSS 提取和压缩
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');

module.exports = {
  plugins: [
    new MiniCssExtractPlugin({
      filename: '[name].[contenthash].css',
      chunkFilename: '[id].[contenthash].css'
    })
  ],
  optimization: {
    minimizer: [
      new CssMinimizerPlugin({
        minimizerOptions: {
          preset: [
            'default',
            {
              discardComments: { removeAll: true },
            },
          ],
        },
      })
    ]
  },
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          MiniCssExtractPlugin.loader,
          'css-loader',
          'postcss-loader'
        ]
      }
    ]
  }
};
```

---

## 7. 缓存策略

### 7.1 文件名哈希

```javascript
module.exports = {
  output: {
    filename: '[name].[contenthash].js',
    chunkFilename: '[name].[contenthash].chunk.js',
    assetModuleFilename: 'assets/[name].[contenthash][ext]'
  },
  optimization: {
    moduleIds: 'deterministic',
    runtimeChunk: 'single',
    splitChunks: {
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
      },
    },
  }
};
```

### 7.2 HTTP 缓存配置

```javascript
// 服务器配置示例 (Express)
app.use('/static', express.static('build/static', {
  maxAge: '1y',
  etag: false
}));

app.use('/', express.static('build', {
  maxAge: '1d',
  etag: true
}));
```

### 7.3 模块级缓存

```javascript
// 持久化缓存
module.exports = {
  cache: {
    type: 'filesystem',
    version: process.env.NODE_ENV,
    cacheDirectory: path.resolve(__dirname, '.webpack-cache'),
    store: 'pack',
    buildDependencies: {
      defaultWebpack: ['webpack/lib/'],
      config: [__filename],
      tsconfig: [path.resolve(__dirname, 'tsconfig.json')],
    },
  }
};
```

---

## 8. 开发与生产环境优化

### 8.1 开发环境优化

```javascript
// webpack.dev.js
module.exports = {
  mode: 'development',
  devtool: 'eval-cheap-module-source-map',
  devServer: {
    hot: true,
    compress: true,
    historyApiFallback: true,
    open: true,
    overlay: {
      warnings: false,
      errors: true
    }
  },
  optimization: {
    removeAvailableModules: false,
    removeEmptyChunks: false,
    splitChunks: false,
  },
  resolve: {
    cacheWithContext: false,
  }
};
```

### 8.2 生产环境优化

```javascript
// webpack.prod.js
const TerserPlugin = require('terser-webpack-plugin');
const CompressionPlugin = require('compression-webpack-plugin');

module.exports = {
  mode: 'production',
  devtool: 'source-map',
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true,
            drop_debugger: true,
            pure_funcs: ['console.log']
          },
          format: {
            comments: false,
          },
        },
        extractComments: false,
      })
    ],
    sideEffects: false,
    usedExports: true,
  },
  plugins: [
    new CompressionPlugin({
      algorithm: 'gzip',
      test: /\.(js|css|html|svg)$/,
      threshold: 8192,
      minRatio: 0.8
    })
  ]
};
```

### 8.3 环境变量优化

```javascript
// webpack.config.js
const webpack = require('webpack');

module.exports = {
  plugins: [
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
      '__DEV__': process.env.NODE_ENV === 'development',
      '__PROD__': process.env.NODE_ENV === 'production'
    })
  ]
};
```

---

## 9. 高级优化技术

### 9.1 模块联邦 (Module Federation)

```javascript
// host 应用
const ModuleFederationPlugin = require('@module-federation/webpack');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        mf_shell: 'shell@http://localhost:3001/remoteEntry.js',
      },
    }),
  ],
};

// remote 应用
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      filename: 'remoteEntry.js',
      exposes: {
        './Button': './src/Button',
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
      },
    }),
  ],
};
```

### 9.2 Web Workers

```javascript
// 主线程
const worker = new Worker(
  new URL('./worker.js', import.meta.url),
  { type: 'module' }
);

worker.postMessage({ data: largeDataSet });

worker.onmessage = (event) => {
  console.log('处理完成:', event.data);
};

// worker.js
self.onmessage = (event) => {
  const result = heavyComputation(event.data);
  self.postMessage(result);
};
```

### 9.3 Progressive Web App (PWA)

```javascript
// webpack.config.js
const WorkboxPlugin = require('workbox-webpack-plugin');

module.exports = {
  plugins: [
    new WorkboxPlugin.GenerateSW({
      clientsClaim: true,
      skipWaiting: true,
      runtimeCaching: [
        {
          urlPattern: /^https:\/\/api\./,
          handler: 'NetworkFirst',
          options: {
            cacheName: 'api-cache',
            networkTimeoutSeconds: 3,
            cacheableResponse: {
              statuses: [0, 200],
            },
          },
        },
      ],
    }),
  ],
};
```

### 9.4 HTTP/2 Server Push

```javascript
// 服务器端实现
app.get('/', (req, res) => {
  // 推送关键资源
  res.push('/static/css/main.css');
  res.push('/static/js/vendor.js');
  res.push('/static/js/main.js');
  
  res.sendFile(path.join(__dirname, 'build/index.html'));
});
```

---

## 10. 监控与调试

### 10.1 性能监控

```javascript
// 性能指标收集
const observer = new PerformanceObserver((list) => {
  list.getEntries().forEach((entry) => {
    console.log(`${entry.name}: ${entry.duration}ms`);
    
    // 发送到监控服务
    analytics.track('performance_metric', {
      name: entry.name,
      duration: entry.duration,
      timestamp: entry.startTime
    });
  });
});

observer.observe({ entryTypes: ['measure', 'navigation'] });
```

### 10.2 错误监控

```javascript
// 全局错误处理
window.addEventListener('error', (event) => {
  console.error('JavaScript错误:', event.error);
  
  // 发送错误报告
  errorReporting.captureException(event.error, {
    extra: {
      filename: event.filename,
      lineno: event.lineno,
      colno: event.colno
    }
  });
});

// Promise 错误处理
window.addEventListener('unhandledrejection', (event) => {
  console.error('未处理的Promise拒绝:', event.reason);
  errorReporting.captureException(event.reason);
});
```

### 10.3 构建分析脚本

```javascript
// analyze-build.js
const fs = require('fs');
const path = require('path');

function analyzeBuildStats(statsPath) {
  const stats = JSON.parse(fs.readFileSync(statsPath, 'utf8'));
  
  const analysis = {
    bundleSize: stats.assets.reduce((total, asset) => total + asset.size, 0),
    chunkCount: stats.chunks.length,
    moduleCount: stats.modules.length,
    duplicateModules: findDuplicateModules(stats.modules),
    largestAssets: stats.assets
      .sort((a, b) => b.size - a.size)
      .slice(0, 10)
  };
  
  console.log('构建分析报告:', JSON.stringify(analysis, null, 2));
  
  // 生成优化建议
  generateOptimizationSuggestions(analysis);
}

function generateOptimizationSuggestions(analysis) {
  const suggestions = [];
  
  if (analysis.bundleSize > 500000) {
    suggestions.push('考虑实施代码拆分以减少包大小');
  }
  
  if (analysis.duplicateModules.length > 0) {
    suggestions.push('发现重复模块，考虑优化依赖结构');
  }
  
  if (analysis.chunkCount > 20) {
    suggestions.push('块数量过多，考虑合并一些较小的块');
  }
  
  console.log('优化建议:', suggestions);
}
```

### 10.4 自动化性能测试

```javascript
// performance-test.js
const puppeteer = require('puppeteer');

async function performanceTest() {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  
  // 启用性能监控
  await page.tracing.start({ path: 'trace.json' });
  
  // 导航到页面
  await page.goto('http://localhost:3000');
  
  // 等待页面加载完成
  await page.waitForLoadState('networkidle');
  
  // 获取性能指标
  const metrics = await page.evaluate(() => {
    return JSON.stringify(performance.getEntriesByType('navigation'));
  });
  
  // 停止跟踪
  await page.tracing.stop();
  
  await browser.close();
  
  // 分析结果
  const navigationTiming = JSON.parse(metrics)[0];
  console.log('页面加载时间:', navigationTiming.loadEventEnd - navigationTiming.navigationStart, 'ms');
  console.log('首次内容绘制:', navigationTiming.responseEnd - navigationTiming.navigationStart, 'ms');
}

performanceTest().catch(console.error);
```

---

## 总结

### 核心优化原则

1. **测量驱动**: 始终基于真实数据进行优化决策
2. **渐进优化**: 从影响最大的优化开始，逐步改进
3. **平衡取舍**: 在构建时间、包大小、运行时性能间找到平衡
4. **持续监控**: 建立持续的性能监控和警报机制

### 实施检查清单

- [ ] 实施 bundle 分析
- [ ] 配置代码拆分
- [ ] 启用 Tree Shaking
- [ ] 优化构建性能
- [ ] 实施缓存策略
- [ ] 优化资源加载
- [ ] 设置性能监控
- [ ] 建立性能预算
- [ ] 自动化性能测试
- [ ] 文档化优化流程

### 性能预算建议

| 指标 | 目标值 | 警告值 |
|------|--------|--------|
| JavaScript 包大小 | < 250KB | > 500KB |
| CSS 包大小 | < 50KB | > 100KB |
| 首次内容绘制 (FCP) | < 1.5s | > 3s |
| 最大内容绘制 (LCP) | < 2.5s | > 4s |
| 累计布局偏移 (CLS) | < 0.1 | > 0.25 |
| 首次输入延迟 (FID) | < 100ms | > 300ms |

通过系统性地应用这些优化策略，可以显著提升 webpack 构建的应用性能，改善用户体验并降低服务器成本。