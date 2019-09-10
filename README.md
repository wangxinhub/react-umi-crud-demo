# React 丨Umi + Dva + Antd CRUD Demo



### 开始之前

- `node 8.4`或以上版本
- 用`cnpm`或`yarn`管理包依赖

### 安装`umi`

```js
yarn global add umi
```

### 创建`umi`项目

```
yarn create umi
```

选择app

```
? Select the boilerplate type (Use arrow keys)
  ant-design-pro  - Create project with an layout-only ant-design-pro boilerplate, use together with umi block.
❯ app             - Create project with a simple boilerplate, support typescript.
  block           - Create a umi block.
  library         - Create a library with umi.
  plugin          - Create a umi plugin.
```

是否选择`TS`?

```
? Do you want to use typescript? (y/N)
N
```

希望启动什么功能?

```
? What functionality do you want to enable?
 (*) antd
>(*) dva
 ( ) code splitting
 ( ) dll
```

根据上述选择，生成文件

```
create package.json
   create .gitignore
   create .editorconfig
   create .env
   create .eslintrc
   create .prettierignore
   create .prettierrc
   create .umirc.js
   create mock\.gitkeep
   create src\app.js
   create src\assets\yay.jpg
   create src\global.css
   create src\layouts\__tests__\index.test.js
   create src\layouts\index.css
   create src\layouts\index.js
   create src\models\.gitkeep
   create src\pages\__tests__\index.test.js
   create src\pages\index.css
   create src\pages\index.js
   create tslint.yml
   create webpack.config.js
✨  File Generate Done
Done in 276.11s.
```

安装依赖

```
yarn
```

启动本地开发环境

```
yarn start
```

如果顺利，在浏览器中输入`localhost:8000`，可以看到下面的页面。
[](https://gw.alipayobjects.com/zos/rmsportal/YIFycZRnWWeXBGnSoFoT.png)

### 配置代理

- 编辑`.umirc.js`

```js
// 配置代理，能通过Restful方式访问
"proxy": {
    "/api": {
        "target": "http://jsonplaceholder.typicode.com/",
        "changeOrigin": true,
        "pathRewrite": { "^/api": "" }
    }
}
```

mock api egg: http://localhost:8000/api/users



### 生成`users`路由

umi 中文件即路由，所以我们要新增路由，新建文件即可!

新增`src/pages/users/index.js`文件，命令：

```
$ umi g page users/index
   create src\pages\users\index.js
   create src\pages\users\index.css
√  success
```

`src/pages/users/index.js`文件内容，如下：

```jsx
import styles from './index.css';
export default function () {
  return (
    <div className={styles.normal}>
      <h1>Page users index</h1>
    </div>
  );
}
```

在浏览器中输入`http://localhost:8000/users`, 会看到`Page users index`的输出。

### 构造`users models`和 `service`

新增 `src/pages/users/models/users.js`，内容如下：

```jsx
import * as usersService from '../services/users';

export default {
  namespace: 'users',
  state: {
    list: [],
    total: null,
  },
  reducers: {
    save(state, { payload: { data: list, total } }) {
      return { ...state, list, total };
    },
  },
  effects: {
    *fetch({ payload: { page } }, { call, put }) {
      const { data, headers } = yield call(usersService.fetch, { page });
      yield put({ type: 'save', payload: { data, total: headers['x-total-count'] } });
    },
  },
  subscriptions: {
    setup({ dispatch, history }) {
      return history.listen(({ pathname, query }) => {
        if (pathname === '/users') {
          dispatch({ type: 'fetch', payload: query });
        }
      });
    },
  },
};
```

新增 `src/pages/users/services/users.js`：

```jsx
import request from '../../../utils/request';

export function fetch({ page = 1 }) {
  return request(`/api/users?_page=${page}&_limit=5`);
}
```

由于我们需要从 response headers 中获取 total users 数量，所以需要改造下 `src/utils/request.js`：

```jsx
import fetch from 'dva/fetch';

function checkStatus(response) {
  if (response.status >= 200 && response.status < 300) {
    return response;
  }
  const error = new Error(response.statusText);
  error.response = response;
  throw error;
}

/**
 * Requests a URL, returning a promise.
 *
 * @param  {string} url       The URL we want to request
 * @param  {object} [options] The options we want to pass to "fetch"
 * @return {object}           An object containing either "data" or "err"
 */
export default async function request(url, options) {
  const response = await fetch(url, options);
  checkStatus(response);
  const data = await response.json();
  const ret = {
    data,
    headers: {},
  };
  if (response.headers.get('x-total-count')) {
    ret.headers['x-total-count'] = response.headers.get('x-total-count');
  }
  return ret;
}
```

再次访问`http://localhost:8000/users`页面的时候，观察一下`Network`, 应该是有`Request URL: http://localhost:8000/api/users?_page=1&_limit=5`, 也有对应的`Response`返回值。说明API接口已经调通了，接来下就开始弄视图层了。



### 添加页面，展示`users`列表

我们把组件存在 src/pages/users/components 里，所以在这里新建 Users.js 和 Users.css。具体参考这个 [Commit](https://github.com/umijs/umi-dva-user-dashboard/commit/ee9ccba33736c31dd8271fabf925c58a9edf76d0)。

需留意两件事：

1. 对 model 进行了微调，加入了 page 表示当前页
2. 由于 components 和 services 中都用到了 pageSize，所以提取到 `src/constants.js`

改完后，切换到浏览器，应该能看到带分页的用户列表。

[](https://camo.githubusercontent.com/2f75960f18fed0f5b5b705e323df8e380138b3b1/68747470733a2f2f7a6f732e616c697061796f626a656374732e636f6d2f726d73706f7274616c2f6763456c70527054446b7055456d72585247486e2e706e67)

### 添加 layout

添加 layout 布局，使得我们可以在首页和用户列表页之间来回切换。umi 里约定 layouts/index.js 为全局路由，所以我们新增 `src/layouts/index.js` 和 CSS 文件即可。

参考这个 [Commit](https://github.com/umijs/umi-dva-user-dashboard/commit/2c763479762ed9fb16e151d362df2b8d68631cd0)。

注意：

1. 页头的菜单会随着页面切换变化，高亮显示当前页所在的菜单项

### 处理 loading 状态

dva 有一个管理 effects 执行的 hook，并基于此封装了 dva-loading 插件。通过这个插件，我们可以不必一遍遍地写 showLoading 和 hideLoading，当发起请求时，插件会自动设置数据里的 loading 状态为 true 或 false 。然后我们在渲染 components 时绑定并根据这个数据进行渲染。

umi-plugin-dva 默认内置了 dva-loading 插件。

然后在 `src/components/Users/Users.js` 里绑定 loading 数据：

```jsx
+ loading: state.loading.models.users,
```

具体参考这个 [Commit](https://github.com/umijs/umi-dva-user-dashboard/commit/f81d36c639d35e67db8b2d0a65b561e4af203eb1) 。

刷新浏览器，你的用户列表有 loading 了没?

### 处理分页

只改一个文件 `src/pages/users/components/Users.js` 就好。

处理分页有两个思路：

1. 发 action，请求新的分页数据，保存到 model，然后自动更新页面
2. 切换路由 (由于之前监听了路由变化，所以后续的事情会自动处理)

我们用的是思路 2 的方式，好处是用户可以直接访问到 page 2 或其他页面。

参考这个 [Commit](https://github.com/umijs/umi-dva-user-dashboard/commit/156220ce72e7d26e2823e8b98fbbd7e5c991db9b) 。

### 删除功能

1. service, 修改 `src/pages/users/services/users.js`：

```jsx
export function remove(id) {
  return request(`/api/users/${id}`, {
    method: 'DELETE',
  });
}
```

1. model, 修改 `src/pages/users/models/users.js`：

```jsx
*remove({ payload: id }, { call, put, select }) {
  yield call(usersService.remove, id);
  const page = yield select(state => state.users.page);
  yield put({ type: 'fetch', payload: { page } });
},
```

1. component, 修改 `src/pages/users/components/Users.js`，替换 `deleteHandler` 内容：

```jsx
dispatch({
  type: 'users/remove',
  payload: id,
});
```

切换到浏览器，删除功能应该已经生效。

### 编辑功能

处理用户编辑和前面的一样，遵循三步走：

1. service
2. model
3. component

先是 service，修改 `src/pages/users/services/users.js`：

```jsx
export function patch(id, values) {
  return request(`/api/users/${id}`, {
    method: 'PATCH',
    body: JSON.stringify(values),
  });
}
```

再是 model，修改 `src/pages/users/models/users.js`：

```jsx
*patch({ payload: { id, values } }, { call, put, select }) {
  yield call(usersService.patch, id, values);
  const page = yield select(state => state.users.page);
  yield put({ type: 'fetch', payload: { page } });
},
```

最后是 component，详见 [Commit](https://github.com/umijs/umi-dva-user-dashboard/commit/6044e2fcaf53ce3645217194abe0fffa7585477c)。

需要注意的一点是，我们在这里如何处理 Modal 的 visible 状态，有几种选择：

1. 存 dva 的 model state 里
2. 存 component state 里

另外，怎么存也是个问题，可以：

1. 只有一个 visible，然后根据用户点选的 user 填不同的表单数据
2. 几个 user 几个 visible

此教程选的方案是 2-2，即存 component state，并且 visible 按 user 存。另外为了使用的简便，封装了一个 `UserModal` 的组件。

完成后，切换到浏览器，应该就能对用户进行编辑了。

### 用户创建

相比用户编辑，用户创建更简单些，因为可以共用 `UserModal` 组件。和编辑比较类似，详见 [Commit](https://github.com/umijs/umi-dva-user-dashboard/commit/91708ca1e39234a4dc07aa7a7193d05358894329) 。

### Github

https://github.com/wangxin1987/react-umi-crud-demo

### 参考资料

```
https://umijs.org/zh/
https://github.com/dvajs/dva
https://github.com/sorrycc/blog/issues/62
```