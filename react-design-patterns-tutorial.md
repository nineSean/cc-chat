# React设计模式完整教程

## 目录

1. [设计模式简介](#设计模式简介)
2. [为什么React开发者需要学习设计模式](#为什么react开发者需要学习设计模式)
3. [设计模式基础](#设计模式基础)
4. [创建型模式](#创建型模式)
   - [工厂模式](#工厂模式)
   - [建造者模式](#建造者模式)
   - [单例模式](#单例模式)
   - [原型模式](#原型模式)
5. [结构型模式](#结构型模式)
   - [适配器模式](#适配器模式)
   - [装饰器模式](#装饰器模式)
   - [外观模式](#外观模式)
   - [代理模式](#代理模式)
   - [组合模式](#组合模式)
6. [行为型模式](#行为型模式)
   - [观察者模式](#观察者模式)
   - [策略模式](#策略模式)
   - [命令模式](#命令模式)
   - [状态模式](#状态模式)
   - [责任链模式](#责任链模式)
7. [React特有模式](#react特有模式)
   - [高阶组件(HOC)](#高阶组件hoc)
   - [Render Props](#render-props)
   - [自定义Hooks模式](#自定义hooks模式)
   - [组合vs继承](#组合vs继承)
   - [受控vs非受控组件](#受控vs非受控组件)
8. [高级模式](#高级模式)
   - [提供者模式](#提供者模式)
   - [复合组件模式](#复合组件模式)
   - [状态缩减器模式](#状态缩减器模式)
9. [最佳实践](#最佳实践)
10. [总结与资源](#总结与资源)

---

## 设计模式简介

设计模式是解决软件设计中常见问题的可重用解决方案。它们是经过验证的最佳实践，能够帮助开发者编写更清晰、更灵活、更可维护的代码。

### 设计模式的核心原则

1. **开闭原则**: 对扩展开放，对修改关闭
2. **单一职责原则**: 一个类应该只有一个改变的理由
3. **依赖倒置原则**: 依赖于抽象，不依赖于具体实现
4. **里氏替换原则**: 子类必须能够替换其基类
5. **接口隔离原则**: 客户端不应该依赖它不需要的接口

---

## 为什么React开发者需要学习设计模式

### 1. 组件复用性
设计模式帮助创建可重用的组件，减少代码重复。

### 2. 代码可维护性
良好的模式使代码结构清晰，便于维护和调试。

### 3. 团队协作
标准化的模式让团队成员更容易理解代码结构。

### 4. 性能优化
某些模式能够帮助优化React应用的性能。

### 5. 架构设计
模式提供了构建大型应用的架构指导。

---

## 设计模式基础

### 模式分类

- **创建型模式**: 处理对象创建
- **结构型模式**: 处理对象组合
- **行为型模式**: 处理对象间的交互

---

## 创建型模式

### 工厂模式

工厂模式用于创建对象而不指定确切的类。

```jsx
// 组件工厂
const ComponentFactory = {
  createButton: (props) => {
    switch (props.type) {
      case 'primary':
        return <PrimaryButton {...props} />;
      case 'secondary':
        return <SecondaryButton {...props} />;
      case 'danger':
        return <DangerButton {...props} />;
      default:
        return <DefaultButton {...props} />;
    }
  },
  
  createInput: (props) => {
    switch (props.type) {
      case 'text':
        return <TextInput {...props} />;
      case 'email':
        return <EmailInput {...props} />;
      case 'password':
        return <PasswordInput {...props} />;
      default:
        return <DefaultInput {...props} />;
    }
  }
};

// 使用工厂
const MyForm = () => {
  return (
    <form>
      {ComponentFactory.createInput({ type: 'email', placeholder: 'Email' })}
      {ComponentFactory.createInput({ type: 'password', placeholder: 'Password' })}
      {ComponentFactory.createButton({ type: 'primary', children: 'Login' })}
    </form>
  );
};
```

### 建造者模式

建造者模式用于构建复杂对象。

```jsx
// 表单建造者
class FormBuilder {
  constructor() {
    this.fields = [];
    this.actions = [];
    this.validation = {};
  }
  
  addField(type, name, props = {}) {
    this.fields.push({ type, name, props });
    return this;
  }
  
  addValidation(field, rules) {
    this.validation[field] = rules;
    return this;
  }
  
  addAction(type, text, handler) {
    this.actions.push({ type, text, handler });
    return this;
  }
  
  build() {
    return (
      <Form 
        fields={this.fields}
        actions={this.actions}
        validation={this.validation}
      />
    );
  }
}

// 使用建造者
const loginForm = new FormBuilder()
  .addField('email', 'email', { placeholder: 'Enter email' })
  .addField('password', 'password', { placeholder: 'Enter password' })
  .addValidation('email', { required: true, email: true })
  .addValidation('password', { required: true, minLength: 6 })
  .addAction('submit', 'Login', handleLogin)
  .addAction('button', 'Cancel', handleCancel)
  .build();
```

### 单例模式

确保一个类只有一个实例。

```jsx
// API客户端单例
class ApiClient {
  constructor() {
    if (ApiClient.instance) {
      return ApiClient.instance;
    }
    
    this.baseURL = process.env.REACT_APP_API_URL;
    this.token = null;
    ApiClient.instance = this;
  }
  
  setToken(token) {
    this.token = token;
  }
  
  async request(endpoint, options = {}) {
    const url = `${this.baseURL}${endpoint}`;
    const config = {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...(this.token && { Authorization: `Bearer ${this.token}` }),
        ...options.headers,
      },
    };
    
    return fetch(url, config);
  }
}

// 使用单例
const api = new ApiClient();

// 在任何地方使用
const UserProfile = () => {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    api.request('/user/profile')
      .then(response => response.json())
      .then(setUser);
  }, []);
  
  return <div>{user?.name}</div>;
};
```

### 原型模式

通过克隆现有对象创建新对象。

```jsx
// 组件配置原型
const baseComponentConfig = {
  theme: 'light',
  size: 'medium',
  disabled: false,
  loading: false,
};

// 克隆并扩展配置
const createButtonConfig = (overrides = {}) => ({
  ...baseComponentConfig,
  type: 'button',
  variant: 'primary',
  ...overrides,
});

const createInputConfig = (overrides = {}) => ({
  ...baseComponentConfig,
  type: 'input',
  variant: 'outlined',
  ...overrides,
});

// 使用原型
const primaryButton = createButtonConfig({ variant: 'primary' });
const dangerButton = createButtonConfig({ variant: 'danger', size: 'large' });
```

---

## 结构型模式

### 适配器模式

适配器模式允许不兼容的接口协同工作。

```jsx
// 旧的API响应格式
const oldApiResponse = {
  user_name: 'John Doe',
  user_email: 'john@example.com',
  user_id: 123,
};

// 新的组件期望的格式
const expectedFormat = {
  name: 'John Doe',
  email: 'john@example.com',
  id: 123,
};

// 适配器
const userDataAdapter = (oldData) => ({
  name: oldData.user_name,
  email: oldData.user_email,
  id: oldData.user_id,
});

// 高阶组件适配器
const withDataAdapter = (WrappedComponent, adapter) => {
  return (props) => {
    const adaptedData = adapter(props.data);
    return <WrappedComponent {...props} data={adaptedData} />;
  };
};

// 使用适配器
const UserCard = ({ data }) => (
  <div>
    <h3>{data.name}</h3>
    <p>{data.email}</p>
  </div>
);

const AdaptedUserCard = withDataAdapter(UserCard, userDataAdapter);
```

### 装饰器模式

装饰器模式动态地给对象添加额外的功能。

```jsx
// 基础组件
const BaseButton = ({ children, ...props }) => (
  <button {...props}>{children}</button>
);

// 装饰器HOCs
const withLoading = (WrappedComponent) => {
  return ({ loading, children, ...props }) => {
    if (loading) {
      return <WrappedComponent {...props}>Loading...</WrappedComponent>;
    }
    return <WrappedComponent {...props}>{children}</WrappedComponent>;
  };
};

const withAnalytics = (WrappedComponent) => {
  return ({ trackingId, ...props }) => {
    const handleClick = (e) => {
      if (trackingId) {
        analytics.track(trackingId, { event: 'button_click' });
      }
      props.onClick?.(e);
    };
    
    return <WrappedComponent {...props} onClick={handleClick} />;
  };
};

const withTooltip = (WrappedComponent) => {
  return ({ tooltip, ...props }) => (
    <div title={tooltip}>
      <WrappedComponent {...props} />
    </div>
  );
};

// 组合装饰器
const EnhancedButton = withTooltip(
  withAnalytics(
    withLoading(BaseButton)
  )
);

// 使用
<EnhancedButton
  loading={isLoading}
  trackingId="login-button"
  tooltip="Click to login"
  onClick={handleLogin}
>
  Login
</EnhancedButton>
```

### 外观模式

外观模式提供统一的接口来访问复杂的子系统。

```jsx
// 复杂的子系统
class ValidationService {
  validateEmail(email) {
    return /\S+@\S+\.\S+/.test(email);
  }
  
  validatePassword(password) {
    return password.length >= 8;
  }
}

class ApiService {
  async login(credentials) {
    return fetch('/api/login', {
      method: 'POST',
      body: JSON.stringify(credentials),
    });
  }
}

class AuthService {
  setToken(token) {
    localStorage.setItem('token', token);
  }
  
  getToken() {
    return localStorage.getItem('token');
  }
}

// 外观类
class AuthFacade {
  constructor() {
    this.validation = new ValidationService();
    this.api = new ApiService();
    this.auth = new AuthService();
  }
  
  async login(email, password) {
    // 验证
    if (!this.validation.validateEmail(email)) {
      throw new Error('Invalid email');
    }
    
    if (!this.validation.validatePassword(password)) {
      throw new Error('Password must be at least 8 characters');
    }
    
    // 登录
    const response = await this.api.login({ email, password });
    const data = await response.json();
    
    // 保存token
    this.auth.setToken(data.token);
    
    return data.user;
  }
}

// React组件中使用外观
const LoginForm = () => {
  const [credentials, setCredentials] = useState({ email: '', password: '' });
  const authFacade = useMemo(() => new AuthFacade(), []);
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      const user = await authFacade.login(credentials.email, credentials.password);
      console.log('Login successful:', user);
    } catch (error) {
      console.error('Login failed:', error.message);
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      {/* 表单字段 */}
    </form>
  );
};
```

### 代理模式

代理模式为其他对象提供代理以控制对它的访问。

```jsx
// 图片懒加载代理
const LazyImage = ({ src, alt, placeholder, ...props }) => {
  const [imageSrc, setImageSrc] = useState(placeholder);
  const [isLoaded, setIsLoaded] = useState(false);
  const imgRef = useRef();
  
  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && !isLoaded) {
          setImageSrc(src);
          setIsLoaded(true);
          observer.disconnect();
        }
      },
      { threshold: 0.1 }
    );
    
    if (imgRef.current) {
      observer.observe(imgRef.current);
    }
    
    return () => observer.disconnect();
  }, [src, isLoaded]);
  
  return (
    <img
      ref={imgRef}
      src={imageSrc}
      alt={alt}
      {...props}
    />
  );
};

// API缓存代理
class CachedApiProxy {
  constructor(apiService) {
    this.apiService = apiService;
    this.cache = new Map();
  }
  
  async get(url, options = {}) {
    const cacheKey = `${url}:${JSON.stringify(options)}`;
    
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey);
    }
    
    const response = await this.apiService.get(url, options);
    this.cache.set(cacheKey, response);
    
    // 设置缓存过期
    setTimeout(() => {
      this.cache.delete(cacheKey);
    }, 5 * 60 * 1000); // 5分钟
    
    return response;
  }
}
```

### 组合模式

组合模式将对象组合成树形结构以表示"部分-整体"的层次结构。

```jsx
// 菜单组合模式
const MenuItem = ({ children, icon, onClick }) => (
  <div className="menu-item" onClick={onClick}>
    {icon && <span className="icon">{icon}</span>}
    <span>{children}</span>
  </div>
);

const MenuGroup = ({ title, children }) => (
  <div className="menu-group">
    <div className="menu-group-title">{title}</div>
    <div className="menu-group-content">{children}</div>
  </div>
);

const Menu = ({ children }) => (
  <div className="menu">{children}</div>
);

// 使用组合
const AppMenu = () => (
  <Menu>
    <MenuGroup title="Dashboard">
      <MenuItem icon="📊" onClick={() => navigate('/dashboard')}>
        Overview
      </MenuItem>
      <MenuItem icon="📈" onClick={() => navigate('/analytics')}>
        Analytics
      </MenuItem>
    </MenuGroup>
    
    <MenuGroup title="Users">
      <MenuItem icon="👤" onClick={() => navigate('/users')}>
        All Users
      </MenuItem>
      <MenuItem icon="➕" onClick={() => navigate('/users/new')}>
        Add User
      </MenuItem>
    </MenuGroup>
    
    <MenuItem icon="⚙️" onClick={() => navigate('/settings')}>
      Settings
    </MenuItem>
  </Menu>
);
```

---

## 行为型模式

### 观察者模式

观察者模式定义对象间的一对多依赖关系。

```jsx
// 事件管理器
class EventManager {
  constructor() {
    this.listeners = {};
  }
  
  subscribe(event, callback) {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event].push(callback);
    
    // 返回取消订阅函数
    return () => {
      this.listeners[event] = this.listeners[event].filter(cb => cb !== callback);
    };
  }
  
  emit(event, data) {
    if (this.listeners[event]) {
      this.listeners[event].forEach(callback => callback(data));
    }
  }
}

// 全局事件管理器
const eventManager = new EventManager();

// 自定义Hook
const useEvent = (event, callback) => {
  useEffect(() => {
    return eventManager.subscribe(event, callback);
  }, [event, callback]);
};

const useEmitEvent = () => {
  return useCallback((event, data) => {
    eventManager.emit(event, data);
  }, []);
};

// 使用示例
const UserProfile = () => {
  const [user, setUser] = useState(null);
  const emit = useEmitEvent();
  
  useEvent('user:updated', (updatedUser) => {
    setUser(updatedUser);
  });
  
  const handleUpdate = (userData) => {
    // 更新用户后通知其他组件
    emit('user:updated', userData);
  };
  
  return <div>{/* 用户界面 */}</div>;
};

const Notification = () => {
  const [message, setMessage] = useState('');
  
  useEvent('user:updated', (user) => {
    setMessage(`用户 ${user.name} 已更新`);
  });
  
  return message ? <div className="notification">{message}</div> : null;
};
```

### 策略模式

策略模式定义一系列算法，并使它们可以互换。

```jsx
// 排序策略
const sortStrategies = {
  name: (a, b) => a.name.localeCompare(b.name),
  date: (a, b) => new Date(a.date) - new Date(b.date),
  price: (a, b) => a.price - b.price,
  rating: (a, b) => b.rating - a.rating,
};

// 过滤策略
const filterStrategies = {
  all: () => true,
  active: (item) => item.status === 'active',
  inactive: (item) => item.status === 'inactive',
  featured: (item) => item.featured === true,
};

// 使用策略的组件
const ProductList = ({ products }) => {
  const [sortBy, setSortBy] = useState('name');
  const [filterBy, setFilterBy] = useState('all');
  
  const processedProducts = useMemo(() => {
    const filtered = products.filter(filterStrategies[filterBy]);
    return filtered.sort(sortStrategies[sortBy]);
  }, [products, sortBy, filterBy]);
  
  return (
    <div>
      <div className="controls">
        <select value={sortBy} onChange={e => setSortBy(e.target.value)}>
          <option value="name">按名称排序</option>
          <option value="date">按日期排序</option>
          <option value="price">按价格排序</option>
          <option value="rating">按评级排序</option>
        </select>
        
        <select value={filterBy} onChange={e => setFilterBy(e.target.value)}>
          <option value="all">所有产品</option>
          <option value="active">活跃产品</option>
          <option value="inactive">非活跃产品</option>
          <option value="featured">特色产品</option>
        </select>
      </div>
      
      <div className="product-grid">
        {processedProducts.map(product => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </div>
  );
};
```

### 命令模式

命令模式将请求封装为对象。

```jsx
// 命令接口
class Command {
  execute() {
    throw new Error('Execute method must be implemented');
  }
  
  undo() {
    throw new Error('Undo method must be implemented');
  }
}

// 具体命令
class AddTodoCommand extends Command {
  constructor(todoList, todo) {
    super();
    this.todoList = todoList;
    this.todo = todo;
  }
  
  execute() {
    this.todoList.add(this.todo);
  }
  
  undo() {
    this.todoList.remove(this.todo.id);
  }
}

class RemoveTodoCommand extends Command {
  constructor(todoList, todoId) {
    super();
    this.todoList = todoList;
    this.todoId = todoId;
    this.removedTodo = null;
  }
  
  execute() {
    this.removedTodo = this.todoList.findById(this.todoId);
    this.todoList.remove(this.todoId);
  }
  
  undo() {
    if (this.removedTodo) {
      this.todoList.add(this.removedTodo);
    }
  }
}

// 命令管理器
class CommandManager {
  constructor() {
    this.history = [];
    this.currentIndex = -1;
  }
  
  execute(command) {
    command.execute();
    
    // 移除当前位置之后的命令
    this.history.splice(this.currentIndex + 1);
    this.history.push(command);
    this.currentIndex++;
  }
  
  undo() {
    if (this.currentIndex >= 0) {
      const command = this.history[this.currentIndex];
      command.undo();
      this.currentIndex--;
    }
  }
  
  redo() {
    if (this.currentIndex < this.history.length - 1) {
      this.currentIndex++;
      const command = this.history[this.currentIndex];
      command.execute();
    }
  }
}

// React中使用命令模式
const TodoApp = () => {
  const [todos, setTodos] = useState([]);
  const commandManager = useRef(new CommandManager());
  
  const todoList = {
    add: (todo) => setTodos(prev => [...prev, todo]),
    remove: (id) => setTodos(prev => prev.filter(t => t.id !== id)),
    findById: (id) => todos.find(t => t.id === id),
  };
  
  const addTodo = (todo) => {
    const command = new AddTodoCommand(todoList, todo);
    commandManager.current.execute(command);
  };
  
  const removeTodo = (id) => {
    const command = new RemoveTodoCommand(todoList, id);
    commandManager.current.execute(command);
  };
  
  const undo = () => commandManager.current.undo();
  const redo = () => commandManager.current.redo();
  
  return (
    <div>
      <div className="toolbar">
        <button onClick={undo}>撤销</button>
        <button onClick={redo}>重做</button>
      </div>
      {/* Todo列表 */}
    </div>
  );
};
```

### 状态模式

状态模式允许对象在内部状态改变时改变它的行为。

```jsx
// 状态接口
class State {
  handle(context) {
    throw new Error('Handle method must be implemented');
  }
}

// 具体状态
class LoadingState extends State {
  handle(context) {
    return (
      <div className="loading">
        <Spinner />
        <p>加载中...</p>
      </div>
    );
  }
}

class SuccessState extends State {
  constructor(data) {
    super();
    this.data = data;
  }
  
  handle(context) {
    return (
      <div className="success">
        <h2>加载成功</h2>
        <DataDisplay data={this.data} />
        <button onClick={() => context.setState(new LoadingState())}>
          刷新
        </button>
      </div>
    );
  }
}

class ErrorState extends State {
  constructor(error) {
    super();
    this.error = error;
  }
  
  handle(context) {
    return (
      <div className="error">
        <h2>加载失败</h2>
        <p>{this.error.message}</p>
        <button onClick={() => context.setState(new LoadingState())}>
          重试
        </button>
      </div>
    );
  }
}

class IdleState extends State {
  handle(context) {
    return (
      <div className="idle">
        <h2>准备加载</h2>
        <button onClick={() => context.setState(new LoadingState())}>
          开始加载
        </button>
      </div>
    );
  }
}

// 状态上下文
const useStateMachine = (initialState) => {
  const [state, setState] = useState(initialState);
  
  const context = {
    setState,
    render: () => state.handle(context),
  };
  
  return context;
};

// 使用状态机
const DataComponent = () => {
  const stateMachine = useStateMachine(new IdleState());
  
  useEffect(() => {
    // 监听状态变化，执行相应操作
    if (stateMachine.state instanceof LoadingState) {
      fetchData()
        .then(data => stateMachine.setState(new SuccessState(data)))
        .catch(error => stateMachine.setState(new ErrorState(error)));
    }
  }, [stateMachine.state]);
  
  return stateMachine.render();
};
```

### 责任链模式

责任链模式为请求创建接收者对象的链。

```jsx
// 处理器抽象类
class Handler {
  constructor() {
    this.nextHandler = null;
  }
  
  setNext(handler) {
    this.nextHandler = handler;
    return handler;
  }
  
  handle(request) {
    if (this.canHandle(request)) {
      return this.process(request);
    }
    
    if (this.nextHandler) {
      return this.nextHandler.handle(request);
    }
    
    return null;
  }
  
  canHandle(request) {
    throw new Error('canHandle method must be implemented');
  }
  
  process(request) {
    throw new Error('process method must be implemented');
  }
}

// 具体处理器
class AuthHandler extends Handler {
  canHandle(request) {
    return !request.user;
  }
  
  process(request) {
    return { error: 'Authentication required', redirect: '/login' };
  }
}

class PermissionHandler extends Handler {
  canHandle(request) {
    return request.user && !request.user.hasPermission(request.resource);
  }
  
  process(request) {
    return { error: 'Permission denied', status: 403 };
  }
}

class ValidationHandler extends Handler {
  canHandle(request) {
    return request.data && !this.validateData(request.data);
  }
  
  process(request) {
    return { error: 'Invalid data', validationErrors: this.getErrors(request.data) };
  }
  
  validateData(data) {
    // 验证逻辑
    return true;
  }
  
  getErrors(data) {
    // 获取错误信息
    return [];
  }
}

class ProcessHandler extends Handler {
  canHandle(request) {
    return true; // 总是可以处理
  }
  
  process(request) {
    // 实际处理业务逻辑
    return { success: true, data: request.data };
  }
}

// 设置责任链
const createHandlerChain = () => {
  const authHandler = new AuthHandler();
  const permissionHandler = new PermissionHandler();
  const validationHandler = new ValidationHandler();
  const processHandler = new ProcessHandler();
  
  authHandler
    .setNext(permissionHandler)
    .setNext(validationHandler)
    .setNext(processHandler);
    
  return authHandler;
};

// React中使用责任链
const useRequestHandler = () => {
  const handlerChain = useMemo(() => createHandlerChain(), []);
  
  return useCallback((request) => {
    return handlerChain.handle(request);
  }, [handlerChain]);
};

const ApiForm = () => {
  const handleRequest = useRequestHandler();
  const [result, setResult] = useState(null);
  
  const onSubmit = (data) => {
    const request = {
      user: getCurrentUser(),
      resource: 'api/data',
      data,
    };
    
    const response = handleRequest(request);
    setResult(response);
  };
  
  return (
    <div>
      {/* 表单组件 */}
      {result && (
        <div className={result.error ? 'error' : 'success'}>
          {result.error || 'Success!'}
        </div>
      )}
    </div>
  );
};
```

---

## React特有模式

### 高阶组件(HOC)

高阶组件是参数为组件，返回值为新组件的函数。

```jsx
// 基础HOC
const withLoading = (WrappedComponent) => {
  return (props) => {
    if (props.loading) {
      return <div>Loading...</div>;
    }
    return <WrappedComponent {...props} />;
  };
};

// 复杂HOC示例
const withAuth = (WrappedComponent, options = {}) => {
  const { redirectTo = '/login', requiredRole = null } = options;
  
  return (props) => {
    const { user, isAuthenticated } = useAuth();
    
    if (!isAuthenticated) {
      return <Redirect to={redirectTo} />;
    }
    
    if (requiredRole && !user.roles.includes(requiredRole)) {
      return <div>Access Denied</div>;
    }
    
    return <WrappedComponent {...props} user={user} />;
  };
};

// 数据获取HOC
const withData = (WrappedComponent, dataSource) => {
  return (props) => {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);
    
    useEffect(() => {
      setLoading(true);
      dataSource(props)
        .then(setData)
        .catch(setError)
        .finally(() => setLoading(false));
    }, []);
    
    return (
      <WrappedComponent
        {...props}
        data={data}
        loading={loading}
        error={error}
      />
    );
  };
};

// 组合HOC
const enhance = compose(
  withAuth({ requiredRole: 'admin' }),
  withData(fetchUserData),
  withLoading
);

const UserDashboard = enhance(({ data, user }) => (
  <div>
    <h1>Welcome, {user.name}</h1>
    <UserList users={data} />
  </div>
));
```

### Render Props

Render Props是指一种在React组件之间使用函数prop共享代码的技术。

```jsx
// 基础Render Props
const MouseTracker = ({ render }) => {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMouseMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    
    window.addEventListener('mousemove', handleMouseMove);
    return () => window.removeEventListener('mousemove', handleMouseMove);
  }, []);
  
  return render(position);
};

// 使用Render Props
const App = () => (
  <MouseTracker
    render={({ x, y }) => (
      <div>
        <h1>Move the mouse around!</h1>
        <p>The current mouse position is ({x}, {y})</p>
      </div>
    )}
  />
);

// 复杂的Render Props示例
const DataProvider = ({ url, render }) => {
  const [state, setState] = useState({
    data: null,
    loading: true,
    error: null,
  });
  
  useEffect(() => {
    fetch(url)
      .then(response => response.json())
      .then(data => setState({ data, loading: false, error: null }))
      .catch(error => setState({ data: null, loading: false, error }));
  }, [url]);
  
  return render(state);
};

// 使用
const UserList = () => (
  <DataProvider
    url="/api/users"
    render={({ data, loading, error }) => {
      if (loading) return <div>Loading...</div>;
      if (error) return <div>Error: {error.message}</div>;
      return (
        <ul>
          {data.map(user => (
            <li key={user.id}>{user.name}</li>
          ))}
        </ul>
      );
    }}
  />
);

// Function as Children模式
const Toggle = ({ children }) => {
  const [on, setOn] = useState(false);
  const toggle = () => setOn(!on);
  
  return children({ on, toggle });
};

// 使用
const ToggleExample = () => (
  <Toggle>
    {({ on, toggle }) => (
      <div>
        <button onClick={toggle}>
          {on ? 'Turn off' : 'Turn on'}
        </button>
        {on && <div>The toggle is on!</div>}
      </div>
    )}
  </Toggle>
);
```

### 自定义Hooks模式

自定义Hooks让你在不增加组件的情况下复用状态逻辑。

```jsx
// 基础自定义Hook
const useCounter = (initialValue = 0) => {
  const [count, setCount] = useState(initialValue);
  
  const increment = useCallback(() => setCount(c => c + 1), []);
  const decrement = useCallback(() => setCount(c => c - 1), []);
  const reset = useCallback(() => setCount(initialValue), [initialValue]);
  
  return { count, increment, decrement, reset };
};

// 复杂的自定义Hook
const useApi = (url, options = {}) => {
  const [state, setState] = useState({
    data: null,
    loading: true,
    error: null,
  });
  
  const fetchData = useCallback(async () => {
    try {
      setState(prev => ({ ...prev, loading: true }));
      const response = await fetch(url, options);
      const data = await response.json();
      setState({ data, loading: false, error: null });
    } catch (error) {
      setState({ data: null, loading: false, error });
    }
  }, [url, options]);
  
  useEffect(() => {
    fetchData();
  }, [fetchData]);
  
  return { ...state, refetch: fetchData };
};

// 表单管理Hook
const useForm = (initialValues, validationSchema) => {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});
  
  const setValue = useCallback((name, value) => {
    setValues(prev => ({ ...prev, [name]: value }));
  }, []);
  
  const setFieldTouched = useCallback((name) => {
    setTouched(prev => ({ ...prev, [name]: true }));
  }, []);
  
  const validate = useCallback(() => {
    const newErrors = {};
    
    Object.keys(validationSchema).forEach(field => {
      const rules = validationSchema[field];
      const value = values[field];
      
      rules.forEach(rule => {
        if (!rule.test(value)) {
          newErrors[field] = rule.message;
        }
      });
    });
    
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  }, [values, validationSchema]);
  
  const handleChange = useCallback((e) => {
    const { name, value } = e.target;
    setValue(name, value);
  }, [setValue]);
  
  const handleBlur = useCallback((e) => {
    const { name } = e.target;
    setFieldTouched(name);
  }, [setFieldTouched]);
  
  const handleSubmit = useCallback((onSubmit) => (e) => {
    e.preventDefault();
    
    // 标记所有字段为已触摸
    const allTouched = Object.keys(values).reduce((acc, key) => {
      acc[key] = true;
      return acc;
    }, {});
    setTouched(allTouched);
    
    if (validate()) {
      onSubmit(values);
    }
  }, [values, validate]);
  
  return {
    values,
    errors,
    touched,
    handleChange,
    handleBlur,
    handleSubmit,
    setValue,
    validate,
  };
};

// 使用自定义Hook
const LoginForm = () => {
  const { values, errors, touched, handleChange, handleBlur, handleSubmit } = useForm(
    { email: '', password: '' },
    {
      email: [
        { test: (value) => value.length > 0, message: 'Email is required' },
        { test: (value) => /\S+@\S+\.\S+/.test(value), message: 'Email is invalid' },
      ],
      password: [
        { test: (value) => value.length >= 6, message: 'Password must be at least 6 characters' },
      ],
    }
  );
  
  const onSubmit = (data) => {
    console.log('Form submitted:', data);
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input
          name="email"
          type="email"
          value={values.email}
          onChange={handleChange}
          onBlur={handleBlur}
          placeholder="Email"
        />
        {touched.email && errors.email && (
          <div className="error">{errors.email}</div>
        )}
      </div>
      
      <div>
        <input
          name="password"
          type="password"
          value={values.password}
          onChange={handleChange}
          onBlur={handleBlur}
          placeholder="Password"
        />
        {touched.password && errors.password && (
          <div className="error">{errors.password}</div>
        )}
      </div>
      
      <button type="submit">Login</button>
    </form>
  );
};
```

### 组合vs继承

React推荐使用组合而非继承来实现组件间的代码复用。

```jsx
// 使用组合的容器组件
const Card = ({ header, children, footer, className = '' }) => (
  <div className={`card ${className}`}>
    {header && <div className="card-header">{header}</div>}
    <div className="card-body">{children}</div>
    {footer && <div className="card-footer">{footer}</div>}
  </div>
);

// 特殊用途的Card组件
const UserCard = ({ user, onEdit, onDelete }) => (
  <Card
    header={<h3>{user.name}</h3>}
    footer={
      <div className="actions">
        <button onClick={() => onEdit(user)}>Edit</button>
        <button onClick={() => onDelete(user.id)}>Delete</button>
      </div>
    }
    className="user-card"
  >
    <p>Email: {user.email}</p>
    <p>Role: {user.role}</p>
  </Card>
);

// 使用插槽(slots)的模式
const Dialog = ({ 
  title, 
  children, 
  actions, 
  isOpen, 
  onClose,
  size = 'medium' 
}) => {
  if (!isOpen) return null;
  
  return (
    <div className="dialog-overlay" onClick={onClose}>
      <div 
        className={`dialog dialog-${size}`}
        onClick={e => e.stopPropagation()}
      >
        <div className="dialog-header">
          <h2>{title}</h2>
          <button onClick={onClose}>×</button>
        </div>
        
        <div className="dialog-content">
          {children}
        </div>
        
        {actions && (
          <div className="dialog-actions">
            {actions}
          </div>
        )}
      </div>
    </div>
  );
};

// 使用Dialog
const ConfirmDialog = ({ isOpen, onClose, onConfirm, message }) => (
  <Dialog
    title="Confirm Action"
    isOpen={isOpen}
    onClose={onClose}
    actions={
      <>
        <button onClick={onClose}>Cancel</button>
        <button onClick={onConfirm} className="primary">Confirm</button>
      </>
    }
  >
    <p>{message}</p>
  </Dialog>
);

// 组合多个组件的复杂示例
const Layout = ({ 
  header, 
  sidebar, 
  children, 
  footer,
  sidebarCollapsed = false 
}) => (
  <div className="layout">
    {header && <header className="layout-header">{header}</header>}
    
    <div className="layout-main">
      {sidebar && (
        <aside className={`layout-sidebar ${sidebarCollapsed ? 'collapsed' : ''}`}>
          {sidebar}
        </aside>
      )}
      
      <main className="layout-content">
        {children}
      </main>
    </div>
    
    {footer && <footer className="layout-footer">{footer}</footer>}
  </div>
);

// 使用Layout
const App = () => {
  const [sidebarCollapsed, setSidebarCollapsed] = useState(false);
  
  return (
    <Layout
      header={
        <AppHeader 
          onToggleSidebar={() => setSidebarCollapsed(!sidebarCollapsed)}
        />
      }
      sidebar={<Navigation />}
      footer={<AppFooter />}
      sidebarCollapsed={sidebarCollapsed}
    >
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/users" element={<UserList />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Layout>
  );
};
```

### 受控vs非受控组件

```jsx
// 受控组件
const ControlledInput = ({ value, onChange, ...props }) => (
  <input value={value} onChange={onChange} {...props} />
);

// 非受控组件
const UncontrolledInput = ({ defaultValue, ...props }) => {
  const ref = useRef();
  
  const getValue = () => ref.current.value;
  
  useImperativeHandle(props.ref, () => ({
    getValue,
    focus: () => ref.current.focus(),
  }));
  
  return <input ref={ref} defaultValue={defaultValue} {...props} />;
};

// 混合模式 - 既可以受控也可以非受控
const FlexibleInput = ({ 
  value: controlledValue, 
  defaultValue, 
  onChange,
  ...props 
}) => {
  const [internalValue, setInternalValue] = useState(defaultValue || '');
  const isControlled = controlledValue !== undefined;
  
  const value = isControlled ? controlledValue : internalValue;
  
  const handleChange = (e) => {
    const newValue = e.target.value;
    
    if (!isControlled) {
      setInternalValue(newValue);
    }
    
    onChange?.(e);
  };
  
  return <input value={value} onChange={handleChange} {...props} />;
};

// 自定义Hook支持受控/非受控模式
const useControllableState = (controlledValue, defaultValue, onChange) => {
  const [internalValue, setInternalValue] = useState(defaultValue);
  const isControlled = controlledValue !== undefined;
  
  const value = isControlled ? controlledValue : internalValue;
  
  const setValue = (newValue) => {
    if (!isControlled) {
      setInternalValue(newValue);
    }
    onChange?.(newValue);
  };
  
  return [value, setValue];
};

// 使用自定义Hook的组件
const Toggle = ({ checked: controlledChecked, defaultChecked, onChange }) => {
  const [checked, setChecked] = useControllableState(
    controlledChecked,
    defaultChecked,
    onChange
  );
  
  return (
    <button
      className={`toggle ${checked ? 'checked' : ''}`}
      onClick={() => setChecked(!checked)}
    >
      {checked ? 'ON' : 'OFF'}
    </button>
  );
};
```

---

## 高级模式

### 提供者模式

提供者模式使用React Context在组件树中共享数据。

```jsx
// 主题提供者
const ThemeContext = createContext();

const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState('light');
  
  const toggleTheme = () => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  };
  
  const value = {
    theme,
    setTheme,
    toggleTheme,
    colors: theme === 'light' ? lightColors : darkColors,
  };
  
  return (
    <ThemeContext.Provider value={value}>
      <div className={`app-theme-${theme}`}>
        {children}
      </div>
    </ThemeContext.Provider>
  );
};

const useTheme = () => {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
};

// 认证提供者
const AuthContext = createContext();

const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    // 检查登录状态
    checkAuthStatus()
      .then(setUser)
      .finally(() => setLoading(false));
  }, []);
  
  const login = async (credentials) => {
    const user = await authService.login(credentials);
    setUser(user);
    return user;
  };
  
  const logout = () => {
    authService.logout();
    setUser(null);
  };
  
  const value = {
    user,
    loading,
    isAuthenticated: !!user,
    login,
    logout,
  };
  
  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
};

const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};

// 组合多个提供者
const AppProviders = ({ children }) => (
  <ThemeProvider>
    <AuthProvider>
      <Router>
        <QueryClient>
          {children}
        </QueryClient>
      </Router>
    </AuthProvider>
  </ThemeProvider>
);
```

### 复合组件模式

复合组件模式创建一组协同工作的组件。

```jsx
// Accordion复合组件
const AccordionContext = createContext();

const Accordion = ({ children, allowMultiple = false }) => {
  const [openItems, setOpenItems] = useState(new Set());
  
  const toggle = (item) => {
    setOpenItems(prev => {
      const newSet = new Set(prev);
      if (newSet.has(item)) {
        newSet.delete(item);
      } else {
        if (!allowMultiple) {
          newSet.clear();
        }
        newSet.add(item);
      }
      return newSet;
    });
  };
  
  const value = {
    openItems,
    toggle,
    isOpen: (item) => openItems.has(item),
  };
  
  return (
    <AccordionContext.Provider value={value}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
};

const AccordionItem = ({ children, value }) => {
  const { isOpen } = useContext(AccordionContext);
  
  return (
    <div className={`accordion-item ${isOpen(value) ? 'open' : ''}`}>
      {children}
    </div>
  );
};

const AccordionHeader = ({ children, value }) => {
  const { toggle } = useContext(AccordionContext);
  
  return (
    <button 
      className="accordion-header"
      onClick={() => toggle(value)}
    >
      {children}
    </button>
  );
};

const AccordionPanel = ({ children, value }) => {
  const { isOpen } = useContext(AccordionContext);
  
  if (!isOpen(value)) return null;
  
  return (
    <div className="accordion-panel">
      {children}
    </div>
  );
};

// 使用复合组件
const FAQ = () => (
  <Accordion allowMultiple>
    <AccordionItem value="item1">
      <AccordionHeader value="item1">
        What is React?
      </AccordionHeader>
      <AccordionPanel value="item1">
        React is a JavaScript library for building user interfaces.
      </AccordionPanel>
    </AccordionItem>
    
    <AccordionItem value="item2">
      <AccordionHeader value="item2">
        How do I learn React?
      </AccordionHeader>
      <AccordionPanel value="item2">
        Start with the official React documentation and build projects.
      </AccordionPanel>
    </AccordionItem>
  </Accordion>
);

// 更复杂的复合组件 - Dropdown
const DropdownContext = createContext();

const Dropdown = ({ children, onSelect }) => {
  const [isOpen, setIsOpen] = useState(false);
  const [selectedValue, setSelectedValue] = useState(null);
  
  const select = (value) => {
    setSelectedValue(value);
    setIsOpen(false);
    onSelect?.(value);
  };
  
  const value = {
    isOpen,
    setIsOpen,
    selectedValue,
    select,
  };
  
  return (
    <DropdownContext.Provider value={value}>
      <div className="dropdown">{children}</div>
    </DropdownContext.Provider>
  );
};

const DropdownTrigger = ({ children }) => {
  const { isOpen, setIsOpen, selectedValue } = useContext(DropdownContext);
  
  return (
    <button 
      className="dropdown-trigger"
      onClick={() => setIsOpen(!isOpen)}
    >
      {selectedValue || children}
    </button>
  );
};

const DropdownMenu = ({ children }) => {
  const { isOpen } = useContext(DropdownContext);
  
  if (!isOpen) return null;
  
  return (
    <div className="dropdown-menu">
      {children}
    </div>
  );
};

const DropdownItem = ({ children, value }) => {
  const { select } = useContext(DropdownContext);
  
  return (
    <div 
      className="dropdown-item"
      onClick={() => select(value)}
    >
      {children}
    </div>
  );
};
```

### 状态缩减器模式

使用useReducer管理复杂状态逻辑。

```jsx
// Todo状态管理
const todoReducer = (state, action) => {
  switch (action.type) {
    case 'ADD_TODO':
      return {
        ...state,
        todos: [...state.todos, {
          id: Date.now(),
          text: action.payload,
          completed: false,
        }],
      };
      
    case 'TOGGLE_TODO':
      return {
        ...state,
        todos: state.todos.map(todo =>
          todo.id === action.payload
            ? { ...todo, completed: !todo.completed }
            : todo
        ),
      };
      
    case 'DELETE_TODO':
      return {
        ...state,
        todos: state.todos.filter(todo => todo.id !== action.payload),
      };
      
    case 'SET_FILTER':
      return {
        ...state,
        filter: action.payload,
      };
      
    case 'SET_LOADING':
      return {
        ...state,
        loading: action.payload,
      };
      
    default:
      return state;
  }
};

const initialState = {
  todos: [],
  filter: 'all',
  loading: false,
};

// 自定义Hook封装状态逻辑
const useTodos = () => {
  const [state, dispatch] = useReducer(todoReducer, initialState);
  
  const addTodo = (text) => {
    dispatch({ type: 'ADD_TODO', payload: text });
  };
  
  const toggleTodo = (id) => {
    dispatch({ type: 'TOGGLE_TODO', payload: id });
  };
  
  const deleteTodo = (id) => {
    dispatch({ type: 'DELETE_TODO', payload: id });
  };
  
  const setFilter = (filter) => {
    dispatch({ type: 'SET_FILTER', payload: filter });
  };
  
  const filteredTodos = useMemo(() => {
    switch (state.filter) {
      case 'completed':
        return state.todos.filter(todo => todo.completed);
      case 'active':
        return state.todos.filter(todo => !todo.completed);
      default:
        return state.todos;
    }
  }, [state.todos, state.filter]);
  
  return {
    ...state,
    filteredTodos,
    addTodo,
    toggleTodo,
    deleteTodo,
    setFilter,
  };
};

// 复杂表单状态管理
const formReducer = (state, action) => {
  switch (action.type) {
    case 'SET_FIELD':
      return {
        ...state,
        values: {
          ...state.values,
          [action.field]: action.value,
        },
        errors: {
          ...state.errors,
          [action.field]: null, // 清除错误
        },
      };
      
    case 'SET_ERRORS':
      return {
        ...state,
        errors: action.payload,
      };
      
    case 'SET_TOUCHED':
      return {
        ...state,
        touched: {
          ...state.touched,
          [action.field]: true,
        },
      };
      
    case 'SET_SUBMITTING':
      return {
        ...state,
        isSubmitting: action.payload,
      };
      
    case 'RESET_FORM':
      return action.payload || {
        values: {},
        errors: {},
        touched: {},
        isSubmitting: false,
      };
      
    default:
      return state;
  }
};

const useFormReducer = (initialValues = {}) => {
  const [state, dispatch] = useReducer(formReducer, {
    values: initialValues,
    errors: {},
    touched: {},
    isSubmitting: false,
  });
  
  const setField = (field, value) => {
    dispatch({ type: 'SET_FIELD', field, value });
  };
  
  const setErrors = (errors) => {
    dispatch({ type: 'SET_ERRORS', payload: errors });
  };
  
  const setTouched = (field) => {
    dispatch({ type: 'SET_TOUCHED', field });
  };
  
  const setSubmitting = (isSubmitting) => {
    dispatch({ type: 'SET_SUBMITTING', payload: isSubmitting });
  };
  
  const resetForm = (newValues) => {
    dispatch({ type: 'RESET_FORM', payload: newValues });
  };
  
  return {
    ...state,
    setField,
    setErrors,
    setTouched,
    setSubmitting,
    resetForm,
  };
};
```

---

## 最佳实践

### 1. 选择合适的模式

```jsx
// 根据需求选择模式
const ComponentDecision = () => {
  // 简单的逻辑复用 -> 自定义Hook
  const { count, increment } = useCounter();
  
  // 需要访问组件实例 -> Render Props
  const mousePosition = (
    <MouseTracker render={({ x, y }) => `${x}, ${y}`} />
  );
  
  // 需要包装多个组件 -> HOC
  const EnhancedComponent = withAuth(MyComponent);
  
  // 复杂状态管理 -> Context + Reducer
  const todoApp = <TodoProvider><TodoApp /></TodoProvider>;
  
  return null;
};
```

### 2. 性能优化

```jsx
// 使用React.memo优化
const ExpensiveComponent = React.memo(({ data }) => {
  return <div>{/* 复杂渲染逻辑 */}</div>;
}, (prevProps, nextProps) => {
  // 自定义比较逻辑
  return prevProps.data.id === nextProps.data.id;
});

// 使用useMemo缓存计算结果
const DataProcessor = ({ items }) => {
  const processedData = useMemo(() => {
    return items
      .filter(item => item.active)
      .sort((a, b) => a.priority - b.priority)
      .map(item => ({
        ...item,
        displayName: `${item.name} (${item.category})`,
      }));
  }, [items]);
  
  return <ItemList items={processedData} />;
};

// 使用useCallback缓存函数
const ListContainer = ({ items, onUpdate }) => {
  const handleItemClick = useCallback((item) => {
    onUpdate(item.id, { ...item, clicked: true });
  }, [onUpdate]);
  
  return (
    <div>
      {items.map(item => (
        <Item 
          key={item.id} 
          item={item} 
          onClick={handleItemClick}
        />
      ))}
    </div>
  );
};
```

### 3. 错误处理模式

```jsx
// 错误边界
class ErrorBoundary extends Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }
  
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }
  
  componentDidCatch(error, errorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
    // 发送错误报告
  }
  
  render() {
    if (this.state.hasError) {
      return this.props.fallback || <div>Something went wrong.</div>;
    }
    
    return this.props.children;
  }
}

// 使用错误边界
const App = () => (
  <ErrorBoundary fallback={<ErrorFallback />}>
    <Router>
      <Routes>
        <Route path="/" element={
          <ErrorBoundary fallback={<PageError />}>
            <HomePage />
          </ErrorBoundary>
        } />
      </Routes>
    </Router>
  </ErrorBoundary>
);

// 异步错误处理Hook
const useAsyncError = () => {
  const [, setError] = useState();
  return useCallback((error) => {
    setError(() => {
      throw error;
    });
  }, []);
};

// 安全的异步组件
const SafeAsyncComponent = () => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const throwError = useAsyncError();
  
  const fetchData = async () => {
    try {
      setLoading(true);
      const result = await api.getData();
      setData(result);
    } catch (error) {
      throwError(error);
    } finally {
      setLoading(false);
    }
  };
  
  return (
    <div>
      {loading && <div>Loading...</div>}
      {data && <DataDisplay data={data} />}
      <button onClick={fetchData}>Fetch Data</button>
    </div>
  );
};
```

### 4. 测试友好的模式

```jsx
// 依赖注入模式便于测试
const ApiContext = createContext();

const ApiProvider = ({ children, apiClient = defaultApiClient }) => (
  <ApiContext.Provider value={apiClient}>
    {children}
  </ApiContext.Provider>
);

const useApi = () => {
  const api = useContext(ApiContext);
  if (!api) {
    throw new Error('useApi must be used within ApiProvider');
  }
  return api;
};

// 测试时可以注入mock API
const TestWrapper = ({ children }) => (
  <ApiProvider apiClient={mockApiClient}>
    {children}
  </ApiProvider>
);

// 可测试的组件
const UserProfile = ({ userId }) => {
  const api = useApi();
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    api.getUser(userId).then(setUser);
  }, [api, userId]);
  
  if (!user) return <div>Loading...</div>;
  
  return (
    <div data-testid="user-profile">
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
};
```

### 5. 代码组织模式

```jsx
// 特性文件夹结构
features/
  auth/
    components/
      LoginForm.jsx
      SignupForm.jsx
    hooks/
      useAuth.js
      useLogin.js
    services/
      authService.js
    context/
      AuthContext.js
    index.js
  users/
    components/
    hooks/
    services/
    context/
    index.js

// 导入导出模式
// features/auth/index.js
export { AuthProvider, useAuth } from './context/AuthContext';
export { LoginForm, SignupForm } from './components';
export { useLogin, useSignup } from './hooks';

// 使用
import { AuthProvider, LoginForm } from 'features/auth';
```

---

## 总结与资源

### 设计模式选择指南

| 场景 | 推荐模式 | 原因 |
|------|----------|------|
| 简单状态逻辑复用 | 自定义Hook | 简洁、易测试 |
| 组件功能增强 | HOC | 可组合、可重用 |
| 动态渲染逻辑 | Render Props | 灵活性高 |
| 跨组件数据共享 | Context + Provider | 避免prop drilling |
| 复杂状态管理 | Reducer模式 | 可预测、易调试 |
| 组件协作 | 复合组件 | 关注点分离 |

### 进阶学习资源

1. **官方文档**
   - [React Patterns](https://reactpatterns.com/)
   - [React Beta Docs](https://beta.reactjs.org/)

2. **推荐书籍**
   - "Design Patterns" by Gang of Four
   - "React Design Patterns and Best Practices"
   - "Advanced React Patterns"

3. **实践项目**
   - 构建组件库
   - 状态管理库
   - 复杂表单系统

### 最后的建议

1. **循序渐进**: 从简单模式开始，逐步掌握复杂模式
2. **实践为王**: 在实际项目中应用这些模式
3. **性能优先**: 始终考虑模式对性能的影响
4. **团队协作**: 确保团队成员都理解使用的模式
5. **保持更新**: 关注React生态的最新发展

设计模式不是银弹，但它们是构建可维护、可扩展React应用的重要工具。选择合适的模式，并根据项目需求灵活应用，将大大提升你的开发效率和代码质量。

---

*这份教程涵盖了React开发中最重要的设计模式。持续实践和学习将帮助你成为更好的React开发者。*