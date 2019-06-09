---
layout: post
title: 깃허브 앱 에서 깃허브 API v4 을 사용하기 위한 인증과정
img: github.png
date: 2019-06-09 14:45:00 +0900
description: 깃허브 앱에서 깃허브 API v4 를 사용하기 위한 인증과정 설명
tag: [github, javascript, graphql]
---

> 이 글에서는 깃허브 앱에서 깃허브 API v4 를 사용하기 위한 인증과정을 설명한다.

최근 팀 내에서 간단히 사용할 리뷰 도구를 깃허브 앱의 형태로 만들고 있다.
[프로봇](https://probot.github.io/)을 사용하면 좀 더 쉽게 깃허브 앱을 만들 수 있지만 관리 및 유지보수 측면에서 서버리스 형태로 만들기로 했기 때문에 인증 등 여러 부분을 직접 다루어야 했다.

---

## 깃허브 앱

깃허브 앱이라고 해서 실제 파일이나 코드가 깃허브에 등록되는 것은 아니다.  
별도로 만들어서 외부 서버 등에 위치한 서비스를 깃허브 앱이라는 형태로 등록을 할 수 있는건데, 아래와 같은 이점이 있다.

- 사용자가 손쉽게 본인의 깃허브 저장소에 설치 가능
- OAuth 앱과는 달리 앱이 사용자의 권한을 가지지 않고 별도의 사용자처럼 존재 (봇 이라고 호칭)
- 사용자가 별도로 깃허브 훅을 설정하지 않아도 됨

쉽게말해 개발자가 앱을 만들면 앱의 봇이 생성되고, 사용자가 앱을 설치하면 자신의 저장소에 앱의 봇이 접근할 수 있게 된다.  
깃허브 앱은 깃허브 사용자라면 누구나 생성할 수 있다.

> 깃허브 앱에 관한 참조  
> [깃허브 개발자문서 - 깃허브 앱](https://developer.github.com/apps/about-apps/#about-github-apps)

---

갓허브 앱으로 등록한 서비스는 깃허브에서 제공하는 API 를 통해 깃허브에 접근할 수 있는데, 현재는 [REST API v3](https://developer.github.com/v3/)와 [GraphQL API v4](https://developer.github.com/v4/) 두 가지를 제공하고 있다.

> GraphQL 에 관한 참조  
> [벨로퍼트님의 GraphQL 강좌 1 편: GraphQL 이 무엇인가?](https://velopert.com/2318)  
> [GraphQL vs. REST](https://blog.apollographql.com/graphql-vs-rest-5d425123e34b)

REST API v3 로도 정보를 가져올 수 있지만, 일부 기능 (풀리퀘스트에 리뷰어를 추가하는 기능 등)은 제공되지 않는다.  
반면 GraphQL API v4 에는 [`requestReviews`](https://developer.github.com/v4/mutation/requestreviews/) 뮤테이션 등 더 많은 기능을 제공한다.

> GraphQL 에서는 원하는 정보를 가져오는 액션을 **쿼리**, 변경을 가하는 액션을 **뮤테이션** 이라고 한다

기능을 확인한 후 개발하려고 하니 혼란스웠던 부분은 인증이다.  
깃허브 앱에서 깃허브 API 를 사용하려면 먼저 액세스 토큰을 얻어야 하는데, 액세스 토큰은 REST API v3 로만 발급받을 수 있다. 따라서 **깃허브 앱에서 GraphQL API v4 를 사용하려면, 먼저 REST API v3 를 사용해야 한다**.

깃허브 앱에서 GraphQL API v4 를 사용하는 과정을 Node.JS 를 사용한 예제와 함께 설명하면 다음과 같다.

1.  앱 ID 와 개인키를 이용해 JWT 생성

    ```js
    // jwt 를 만들기 위해 jsonwebtoken 패키지 사용
    import jwt from "jsonwebtoken";

    const TOKEN_EXPIRE_SEC = 60 * 10;

    const createToken = () => {
      const now = Math.floor(Date.now() / 1000);
      const payload = {
        iat: now,
        exp: now + TOKEN_EXPIRE_SEC,
        iss: APP_ID
      };
      return jwt.sign(payload, PRIVATE_KEY, { algorithm: "RS256" });
    };
    ```

    > 앱 ID 와 개인키는 깃허브 앱의 설정 페이지에서 찾을 수 있다.

    > JWT 는 JSON Web Token 의 약자로, JSON 으로 이루어진 정보를 개인키로 암호화해서 사용하는 토큰이다.  
    > 자세한 정보는 [JWT 설명 사이트](https://jwt.io/), [벨로퍼트님의 블로그](https://velopert.com/2389) 참조

2.  생성한 JWT 와 인스톨레이션 ID 를 가지고 REST API v3 를 사용해 액세스 토큰 발급

    ```js
    // 깃허브의 REST API v3 를 사용하기 위한 외부 패키지
    // fetch 나 axios 등 다른 REST client 를 사용해도 무방
    import { request } from "@octokit/request";

    // 깃허브 엔터프라이즈 환경이라면 BASE_URL 변경 필요
    const BASE_URL = "https://api.github.com";

    const restClient = request.defaults({
      baseUrl: BASE_URL,
      headers: {
        accept: "application/vnd.github.machine-man-preview+json"
      }
    });

    const restRequest = async (uri, opt = {}) => {
      const token = await createToken();
      const { data } = await restClient(uri, {
        ...opt,
        headers: {
          ...opt.headers,
          authorization: `Bearer ${token}`
        }
      });

      return data;
    };

    const createAccessToken = async installationId => {
      const { token, expires_at } = await restRequest(
        "POST /v3/app/installations/:installationId/access_tokens",
        {
          installationId
        }
      );

      return {
        token,
        expires_at
      };
    };
    ```

    > 깃허브 앱은 앱 자체 정보만으로 액세스 토큰을 얻을 수 없다. 추가로 인스톨레이션 ID 라는 값이 필요한데, 앱이 특정 사용자 혹은 조직에게 설치되면 생성되는 ID 값이다. 이 값은 깃허브 훅과 함께 전달받거나, API 를 통해 조회할 수 있다.

3.  발급받은 액세스 토큰으로 GraphQL API v4 사용
    ```js
    // 깃허브의 GraphQL API v4 를 사용하기 위한 외부 패키지
    // 다른 GraphQL 클라이언트를 사용해도 무방
    import graphql from "@octokit/graphql";

    const graphqlClient = graphql.defaults({
      baseUrl: BASE_URL
    });
    
    const graphqlRequest = async (installationId, query, opt = {}) => {
      const token = await createAccessToken(installationId);
    
      return await graphqlClient(query, {
        ...opt,
        headers: {
          ...opt.headers,
          authorization: `Bearer ${token}`
        }
      });
    };
    
    const getLoginInfo = async installationId => {
      const query = `query {
        viewer {
          login
        }
      }`;
    
      return await graphqlRequest(installationId, query);
    };
    ```
 
이렇게 정의하면 간단한 함수 호출로 GraphQL API v4 사용이 가능하다.
```js
const main = async () => {
  const INSTALLATION_ID = 27;
  const data = await await getLoginInfo(INSTALLATION_ID);
  
  console.log(data);
};

main();
```

  
위 예제에서는 간결함을 위해 넣지 않았지만, 실제로는 매번 토큰을 발급받지 않고 한 번 발급받은 토큰은 저장해두었다가 만료될때까지 계속해서 사용하는 등의 처리가 필요하다.
