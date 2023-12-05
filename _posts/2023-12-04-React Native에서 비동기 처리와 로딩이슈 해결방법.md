---
layout: page
title: React/React Native에서 비동기 처리와 로딩이슈 해결방법
category: develop
---

> 해당 글은 개발도중 겪었던 이슈를 기반으로 제 나름대로 작성한 글입니다. 정확하지 않을 가능성이 있음을 알려드립니다.

React와 React Native는 JavaScript 기반의 라이브러리로, 비동기적인 특성을 갖고 있습니다. 이는 때때로 예상치 못한 동작이나 로딩 이슈를 초래할 수 있습니다. 이 글에서는 이러한 비동기 처리와 관련된 일반적인 문제들과 그 해결 방안을 살펴보겠습니다.

### API를 통한 데이터 로딩

React에서 가장 흔한 시나리오 중 하나는 API를 통해 데이터를 불러와 화면에 표시하는 것입니다. 이 과정에서 첫 로딩 시, 데이터가 아직 도착하지 않았을 때 undefined, null 또는 다른 더미 데이터가 잠깐 보일 수 있습니다. 이러한 현상은 데이터 로딩이 완료되기 전에 컴포넌트가 렌더링되기 때문에 발생합니다. 이는 React가 데이터를 비동기 처리를 하기에 발생합니다.

```jsx
import React, { useState, useEffect } from "react";

const SampleComponent = () => {
  const [data, setData] = useState(); // 초기 상태는 undefined
  const fetchData = async () => {
    try {
      const response = await fetch("https://some-api.com/data");
      const jsonData = await response.json();
      setData(jsonData);
    } catch (error) {
      console.error("Error fetching data:", error);
    }
  };

  useEffect(() => {
    fetchData();
  }, []);

  return <div>{data ? <div>{data.name}</div> : <div>Loading...</div>}</div>;
};

export default SampleComponent;
```

위 코드에서는 `useEffect`를 사용하여 컴포넌트가 마운트될 때 데이터를 가져오도록 설정했습니다. `data`가 존재하면 데이터를 표시하고, 그렇지 않으면 'Loading...'을 표시합니다. 쉽게 말하면, 데이터를 가져오고 나서 데이터를 표시합니다.

### 다수의 State 사용

여러 개의 `State`를 사용할 때, 하나의 `State`가 다른 `State`에 의존하는 경우가 있습니다. 이럴 때는 각 `State`의 업데이트 순서와 의존성을 주의 깊게 관리해야 합니다. 업데이트 순서를 제대로 관리하지 않을경우, 개발자의 의도와는 다르게 로직이 작동하는 경우가 발생합니다.

```jsx
import React, { useState, useEffect } from "react";

const MultiStateComponent = () => {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    const fetchUserData = async () => {
      try {
        const userResponse = await fetch("https://some-api.com/user");
        const userData = await userResponse.json();
        setUser(userData);

        const postsResponse = await fetch(
          `https://some-api.com/posts?user=${userData.id}`
        );
        const postsData = await postsResponse.json();
        setPosts(postsData);
      } catch (error) {
        console.error("Error fetching data:", error);
      }
    };

    fetchUserData();
  }, []);

  return (
    <div>
      {user && <div>Welcome, {user.name}</div>}
      {posts.length > 0 ? (
        <ul>
          {posts.map((post) => (
            <li key={post.id}>{post.title}</li>
          ))}
        </ul>
      ) : (
        <div>No posts available.</div>
      )}
    </div>
  );
};

export default MultiStateComponent;
```

위의 코드는 user의 데이터를 불러오고 나서, 해당 유저가 작성한 게시글들을 가져와 게시글의 제목을 출력하는 코드 입니다. 간단한 코드이지만, `state`간의 관계를 정의하는게 중요하다라는 것을 알수 있습니다. 코드를 보면, 게시글을 가져오는 api에는 user의 아이디가 필요합니다. 그렇기에 user 데이터를 api로 호출하고, 그 다음에 user의 아이디를 이용하여 유저가 작성한 게시글들을 가져 옵니다. 이때 user 데이터를 가져오는 api와 게시글들을 가져오는 api 둘중에 어느쪽이 먼저 실행될지는 알수 없습니다. 그렇기에 state를 기준으로 먼저 user의 데이터가 가져오도록 api호출시에만 동기적으로 작동하도록 설정하여 `(async, await)` 먼저 user 데이터를 가져올수 있도록 코드를 작성합니다.

이외에도 `useEffect` 훅을 사용하여, userData가 불러와 지면, 게시글을 불러오는 api가 실행되도록하는 방법도 있습니다.

```jsx
import React, { useState, useEffect } from "react";

const MultiStateComponent = () => {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);

  // 사용자 데이터를 불러오는 useEffect
  useEffect(() => {
    const fetchUserData = async () => {
      try {
        const response = await fetch("https://some-api.com/user");
        const userData = await response.json();
        setUser(userData);
      } catch (error) {
        console.error("Error fetching user data:", error);
      }
    };

    fetchUserData();
  }, []);

  // 게시물 데이터를 불러오는 useEffect
  useEffect(() => {
    const fetchPostsData = async () => {
      if (user) {
        try {
          const response = await fetch(
            `https://some-api.com/posts?user=${user.id}`
          );
          const postsData = await response.json();
          setPosts(postsData);
        } catch (error) {
          console.error("Error fetching posts:", error);
        }
      }
    };

    fetchPostsData();
  }, [user]); // user 상태가 바뀌었을 때만 실행

  return (
    <div>
      {user && <div>Welcome, {user.name}</div>}
      {posts.length > 0 ? (
        <ul>
          {posts.map((post) => (
            <li key={post.id}>{post.title}</li>
          ))}
        </ul>
      ) : (
        <div>No posts available.</div>
      )}
    </div>
  );
};

export default MultiStateComponent;
```

이 코드에서는 두 번째 `useEffect`가 첫 번째 useEffect에서 설정한 user `State`에 의존합니다. 이를 통해 데이터의 의존성을 관리하고, 비동기 처리를 효율적으로 수행할 수 있습니다.

React와 React Native에서 비동기 처리를 효과적으로 관리하는 것은 사용자 경험을 개선하는 데 매우 중요합니다. async, await 문법과 useEffect를 활용하여 데이터의 로딩 상태를 관리하고, State 간의 의존성을 적절히 설정함으로써, 더 나은 UI와 사용자 경험을 제공할 수 있습니다.

- 같이보면 좋을 내용

[https://react.dev/learn/lifecycle-of-reactive-effects](https://react.dev/learn/lifecycle-of-reactive-effects)
