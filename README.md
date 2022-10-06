# Router 만들어보기

react-router-dom을 설치해서 그냥 사용해봤을 뿐 window.history에 대해서 그다지 깊게 생각해본적이 없다. 프로그래머스 챌린지 할때 SPA를 구현해야하기 때문에 새로고침 없이 페이지를 전환하는데 histroy.pushState를 사용해서 구현했었다. 그때는 이벤트 리스너를 통해서 페이지 전환을 했었다. 비슷하게 하면 될 것 같아서 '생각보다 간단하겠는데?'라고 생각하면서 과제를 시작했다.

## 처음 아이디어

**과제 요구조건**

```ts
ReactDOM.createRoot(container).render(
  <Router>
    <Route path="/" component={<Root />} />
    <Route path="/about" component={<About />} />
  </Router>
);
```

처음 아이디어는 path를 받아서 component를 랜더링하는 Route컴포넌트를 작성하려고 했다. 그래서 코드를 다음과 같이 만들었다.

- **Route.tsx**

```ts
import React, { useEffect } from "react";

interface IRouteProps {
  path: string;
  component: React.ReactElement;
  state?: {};
}

const Route = ({ path, component, state }: IRouteProps) => {
  useEffect(() => window.history.pushState(state, "", path), [path]);
  return component;
};

export default Route;
```

- **Router.tsx**

```ts
import React from "react";

interface IRoutesProps {
  children: React.ReactElement;
}

const Router = ({ children }: IRoutesProps) => {
  return children;
};

export default Router;
```

그리고 어플리케이션의 Router를 만들었다.

```ts
interface IAppRoutersProps {}

const AppRouters = () => {
  return (
    <Router>
      <>
        <Route path="/" component={<Main />} />
        <Route path="/about" component={<About />} />
      </>
    </Router>
  );
};

export default AppRouters;
```

![이미지]()
당연히 안된다.(ㅋㅋㅋㅋㅋㅋㅋㅋㅋ)

### 왜 안될까?

1. AppRouter는 함수니까 위에서 아래로 코드를 읽으면서 Main 컴포넌트를 실행하고 /about으로 pushState를 넣은 다음에 About 컴포넌트를 실행하기 때문에 한 화면에 두 컴포넌트가 동시에 출력된다.
2. useEffect가 실행될 때, Route에 전달된 path를 보고 그냥 history에 밀어 넣기 때문이다.

### 해결하려면?

일단 기본 url은 기본 동작으로 가지고 있어야한다. 최초로 실행시킬 컴포넌트는 '/' 경로에서 그냥 동작하도록 하게 하고 나머지 url에 대해서는 브라우저 주소창에 어떤 url을 입력할 때마다 입력한 경로를 보고 경로에 해당되는 컴포넌트를 실행해야한다.

그래도 아얘 동작을 하지 않는것은 아니기 때문에 일단 여기서부터 수정을 해야겠다. 그런데 어디서부터 시작해야할지 전혀 감이 오지 않는다. 내가 무에서 유를 창조할 수 있었다면 아직도 놀고있지는 않았을 것 같다. 일단 다른 사람들이 작성한 코드와 문서를 레퍼런스로 삼아야겠다.

## 바닐라 자바스크립트에서 SPA는 어떻게 구현할까?

[프로그래머스 프론트앤드 챌린지](https://prgms.tistory.com/113)에서 SPA를 어떻게 구현했는지 다시 찾아보기로했다.

<iframe href="https://stackblitz.com/edit/js-rjuyzc?embed=1&file=App.js"></iframe>

프로그래머스의 블로그에 적혀있는데로 코드를 구현해보았다. 구현을 하면서 핵심이라고 생각한 부분은 다음과 같다.

1. url이 바뀌었는지 그대로인지 알아야한다.
2. url이 바뀌었다는 것을 감지했다면 해당 컴포넌트를 랜더링 해야한다. 이때 새로고침이 일어나지 않아야한다.

### url이 바뀌었는지는 어떻게 알 수 있을까?

```js
function App({ $target }) {
  // 생략
  const { pathname } = window.location;

  if (pathname === "/") {
    new Main({ $target });
  } else if (pathname === "/about") {
    new About({ $target });
  }
}
```

url의 변경 여부는 window.location.pathname이 변경되었는지 살펴보면된다. 참고로 window.location.pathname으로도 url을 변경할 수 있다. 하지만 이렇게 변경을 하면 새로고침이 일어난다.

<iframe href="https://stackblitz.com/edit/js-7wy1gr?embed=1&file=index.js"></iframe>

새로고침이 일어나지 않고 컴포넌트를 변경하기 위해서는 history.pushState를 사용해야한다. pushState는 url을 변경하기만하고 별다른 동작을 하지 않는다. 그래서 위의 코드는 조건문을 사용하여 pathname을 살펴보고 일치하는 url에 대해서 컴포넌트를 랜더링하는 방식으로 SPA를 구현하고 있다. 지금까지 살펴본 내용을 바탕으로 React 프로젝트에 적용해보기로 했다.

- **lib.ts**

```ts
export function changeRouter(pathname: string) {
  return window.history.pushState(null, "", pathname);
}
```

- **Route.tsx**

```ts
import { changeRouter } from "@/lib/lib";
import React, { useEffect, useState } from "react";

interface IRouteProps {
  path: string;
  component: React.ReactElement;
  state?: {};
}

const Route = ({ path, component, state }: IRouteProps) => {
  const [isPath, setIsPath] = useState(false);

  useEffect(() => {
    const { pathname } = window.location;
    if (path === pathname) {
      setIsPath(true);
      changeRouter(path);
    }
  }, [path]);

  return isPath ? component : null;
};

export default Route;
```

로직은 정말 간단하게 Route만 수정했다. pathname이 path와 일치할 때만 component를 랜더링하도록 로직을 수정하였다. 그러자 라우터에 맞는 컴포넌트만 랜더링한다. Yeah!?

## react-router-dom 깃허브 레포지토리 살펴보기

하지만 위의 코드는 동작은 하더라도 요구사항과 많이 동떨어져있다.(아직 갈길이 멀다.)

1. histroy에서 pushState의 맥락을 공유하는 무언가가 없기 때문에 컴포넌트 내부에서 버튼을 눌렀을 때, pushState를 실행시키면 주소창에 url만 변경되고 컴포넌트는 랜더링 되지 않는다.
2. 뒤로 가기, 앞으로 가기가 동작하지 않는다.
3. Router.tsx 컴포넌트가 불필요하다. 그냥 Route만 있어도 라우팅을 동작시킬 수 있다. Router안에 Route를 반드시 쓰도록 강제할 방법도 없다. 하지만 요구 조건은 Router안에 children으로 Route가 들어있어야한다. react-router-dom(v6) 라이브러리를 실제로 사용할 때, 그 사용 조건은 조금 까다롭다.

```tsx
function Router() {
  return (
    <Routes>
      <Route pathname="/" element={<Main />} />
    </Routes>
  );
}

function App() {
  return <Router />;
}

function Root() {
  return (
    <React.StrictMode>
      <BrowserRouter>
        <App />
      </BrowserRouter>
    </React.StrictMode>
  );
}

ReactDOM.render(<Root />, document.getElementById("root"));
```

위의 규칙을 지키지 않고 Route를 단독으로 사용하려고 시도하거나 BrowserRouter가 App을 감싸고 있지 않는 등의 코드를 작성하면 경고 메시지를 출력하면서 어플리케이션이 동작하지 않는다. 내가 생각했을 때는 **그냥 Route만 있으면 될 것 같은데, 왜 이렇게 동작을 하도록 강제했는지** 잘 모르겠다. 과제의 요구 조건을 만족 시키기 위해서는 몇가지 목적을 가지고 코드를 살펴봐야겠다.

1. BrowserRouter는 어떤 역할인가?
2. 왜 Routes로 Route를 감싸야 하는가?
3. 반드시 Routes안에 Route를 쓰게 하도록 강제하는 방법은 무엇일까?

### BrowserRouter는 어떤 역할을 하는가?

react-router의 깃허브 레포지토리를 살펴보면 BrowserRouter 함수를 찾을 수 있다. 결론부터 말하면 BrowserRouter는 Router 컴포넌트를 반환하고 Router 컴포넌트는 NavigationContext.Provider에 감쌓여있는 LocationContext.Provider를 반환한다.

LocationContext와 NavigationContext는 React Context API다. BrowserRouter로 App 컴포넌트를 감싸으면 어플리케이션 전체 Router 상태를 공유하는 Context API가 생성된다.

```ts
export interface BrowserRouterProps {
  basename?: string;
  children?: React.ReactNode;
  window?: Window;
}

/**
 * A `<Router>` for use in web browsers. Provides the cleanest URLs.
 */
export function BrowserRouter({
  basename,
  children,
  window
}: BrowserRouterProps) {
  let historyRef = React.useRef<BrowserHistory>();
  if (historyRef.current == null) {
    historyRef.current = createBrowserHistory({ window, v5Compat: true });
  }

  let history = historyRef.current;
  let [state, setState] = React.useState({
    action: history.action,
    location: history.location
  });

  React.useLayoutEffect(() => history.listen(setState), [history]);

  return (
    <Router
      basename={basename}
      children={children}
      location={state.location}
      navigationType={state.action}
      navigator={history}
    />
  );
}
```

```ts
/**
 * Provides location context for the rest of the app.
 *
 * Note: You usually won't render a <Router> directly. Instead, you'll render a
 * router that is more specific to your environment such as a <BrowserRouter>
 * in web browsers or a <StaticRouter> for server rendering.
 *
 * @see https://reactrouter.com/docs/en/v6/routers/router
 */
export function Router({
  basename: basenameProp = "/",
  children = null,
  location: locationProp,
  navigationType = NavigationType.Pop,
  navigator,
  static: staticProp = false
}: RouterProps): React.ReactElement | null {
  invariant(
    !useInRouterContext(),
    `You cannot render a <Router> inside another <Router>.` +
      ` You should never have more than one in your app.`
  );

  // Preserve trailing slashes on basename, so we can let the user control
  // the enforcement of trailing slashes throughout the app
  let basename = basenameProp.replace(/^\/*/, "/");
  let navigationContext = React.useMemo(
    () => ({ basename, navigator, static: staticProp }),
    [basename, navigator, staticProp]
  );

  if (typeof locationProp === "string") {
    locationProp = parsePath(locationProp);
  }

  let {
    pathname = "/",
    search = "",
    hash = "",
    state = null,
    key = "default"
  } = locationProp;

  let location = React.useMemo(() => {
    let trailingPathname = stripBasename(pathname, basename);

    if (trailingPathname == null) {
      return null;
    }

    return {
      pathname: trailingPathname,
      search,
      hash,
      state,
      key
    };
  }, [basename, pathname, search, hash, state, key]);

  warning(
    location != null,
    `<Router basename="${basename}"> is not able to match the URL ` +
      `"${pathname}${search}${hash}" because it does not start with the ` +
      `basename, so the <Router> won't render anything.`
  );

  if (location == null) {
    return null;
  }

  return (
    <NavigationContext.Provider value={navigationContext}>
      <LocationContext.Provider
        children={children}
        value={{ location, navigationType }}
      />
    </NavigationContext.Provider>
  );
}
```

### 왜 Routes로 Route를 감싸야하는가?

### 반드시 Routes안에 Route를 쓰게 하도록 강제하는 방법은 무엇일까?

## 마무리

바닐라 자바스크립트로 SPA를 구현하는 것은 상반기 기업 과제에서도 진행한적이 있었고, 프로그래머스 프론트앤드 챌린지 과제로도 구현한 적이 있었다. 그런데 왜 계속 할 때마다 잘 모를까? 좀 현타가 온다.