---
layout: post
title:  "GraphQL 잘 사용하기"
future: true
date:   2022-09-04 00:00:00 +0900
categories: GraphQL
---
최근에 회사에 신규 입사자분들이 많이 오셔서 온보딩겸 준비했던 GraphQL 잘 사용하기! 이다. 동료분이 블로그에도 올리면 좋을것 같다고 하셔서 블로그에도 기록차 남기게 되었다. 
# GraphQL
![](/assets/img/graphql-logo.png)


GraphQL(Graph Query Language)은 페이스북에서 만든 쿼리 언어로서, 그래프로 스키마의 연관관계를 표현한다.
## 구성 요소
- 스키마
- 리졸버

## 그래서 어떤점이 좋은가?
- 클라이언트에서 원하는 값만 가져올 수 있다.(`Over-Fetching` 문제 해결)
- REST API에 비해 변경에 유연하게 대처할 수 있다.
    - 예시) user에 새 필드가 추가되었는데 REST API는 user/v1/123 에서 user/v2/123을 새로 만들어줘야 하지만 GraphQL에서는 새 필드만 추가하면 된다.
- Federation 기능(`Under-Fetching` 문제 해결)

## 단점은
- 서버 개발의 복잡성이 커진다.
- 캐싱이 복잡하다.
- 스키마 관리의 리소스가 든다.

# 목표
- graphql을 어떻게 사용하면  `Under-Fetching`, `Over-Fetching` 문제를 잘 해결할 수 있는지 알아본다.

# MSA에서 GraphQL 사용하기

MSA 구조에서는 어떻게 GraphQL을 사용할 수 있을까?
간단하게 생각하면 아래와 같이 요청할 수 있다.

필요한 데이터는 상품의 상세 정보, 리뷰 정보, 리뷰 정보에는 작성한 유저
1. 상품 api에서 상품의 정보를 얻어온다.
2. 상품 id로 리뷰 api에서 리뷰 정보를 얻어온다.
3. 리뷰 id로 유저 api에서 유저 정보를 얻어온다.

![](/assets/img/msa-graphql1.png)

하지만 실제로 그렇게 하면 네트워크 통신을 왕복으로 3번을 반복하고 클라이언트에서 데이터를 합치는 과정도 필요하다. 이러한 문제를 `Under-Fetching`이라 한다.

## Apollo Federation
Apollo Federation을 사용하면 클라이언트가 단일 쿼리로 원하는 데이터를 얻을 수 있다.


하나의 슈퍼 그래프와 여러개의 서브 그래프로 구성되며, 클라이언트는 마치 **모놀리식으로 구성된것 같은 서버에 요청**을 한다.

![](/assets/img/msa-graphql2.png)

> gateway는 **supergraph**라고 칭하고, 뒤의 api들은 **subgraph**라고 칭한다.

### 게이트 웨이로 쿼리를 요청하면 어떻게 될까?
게이트 웨이는 쿼리 요청이 들어오면 아래의 3가지를 수행한다.

1. gateway 에서 쿼리를 validate 한다.
2. **해당하는 타입이 있는 subgraph로 요청**을 보낸다.
3. gateway는 어떻게 쿼리를 보낼지 Query Plan을 만든다.

> Query Plan은 쿼리를 subgraph에 어떤 순서로 요청할지에 대한 실행계획이다. 자세한 내용은 공식 문서에서 확인할 수 있다.

## 예제
예제로 gateway에 쿼리를 요청하면 어떤식으로 요청이 가는지 확인해보자.
스키마는 아래와 같다.

1. product-api(상품)

```graphql
 type Query {
    products: [Product!]!
    product(id: ID!): Product!
}

type Product {
    id: ID!
    title: String!
    price: Int!
}
```

2. product-review-api(리뷰)

```graphql
type Query {
    reviews: [Review!]!
    review(id: ID!): Review!
}

type Review @key(fields: "id") {
    id: ID!
    content: String!
    user: User!
}

extend type Product @key(fields: "id") {
    id: ID! @external
    reviewInfo: [Review!]!
} 

extend type User @key(fields: "id") {
    id: ID! @external
}
```

3. user-api(유저)

```graphql
type Query {
    users: [User!]!
    user(id: ID!): User!
}

type User@key(fields: "id") {
    id: ID!
    name: String!
}
```

- 상품과 리뷰는 1:N 관계이다.
- 리뷰와 유저는 1:1 관계이다.

상품 상세화면이 있다고 생각하고, 아래와 같은 쿼리를 gateway에 요청한다.
```graphql
query GetProduct($id: ID!) {
    product(id: $id) {
        id
        title
        reviewInfo {
            id
            content
            user {
                id
                name
            }
        }
    }
}
```
1. gateway -> product-api

    슈퍼 그래프인 게이트 웨이에서 서브 그래프인 상품 api로 상품 id, title에 대해 요청한다.
    ![](/assets/img/gql-query1.png)

2. gateway -> product-review-api

    게이트 웨이는 상품의 정보(id, typename)로 product-review-api에 리뷰 정보(id, content, user.id)를 요청한다. 
    ![](/assets/img/gql-query2.png)

3. gateway -> user-api

    user의 id, typename으로 user에 대한 정보를 요청한다.(id, name)
    ![](/assets/img/gql-query3.png)

## 여기서 알 수 있는 점
1. id만 필요한 경우, id만 요청하자!

    위의 예제를 보면, subgraph에서 id와 typename 으로 데이터를 요청하는 것을 알 수 있다. 따라서, 리뷰의 경우 user의 id와 typename은 이미 알고 있으므로, id만 요청하는 경우 user에 또 요청할 필요가 없다. 아래 쿼리의 경우 user-api에 요청이 가지 않는다.
    ```graphql
    query GetProduct($id: ID!) {
        product(id: $id) {
            id
            title
            reviewInfo {
                id
                content
                user {
                    id
                }
            }
        }
    }
    ```

    필요하지 않는 데이터도 불러오는걸 `Over-Fetching`이라 하는데, 필요한 데이터만 쿼리에 추가함으로써 `Over-Fetching`을 막을 수 있다.

    **결론은 꼭 필요한 필드만 쿼리에 넣자!!**

## 그 외 운영 Tip
- gateway는 subgraph의 response를 조합할때 inner join과 비슷한 방식으로 동작한다.
  - 예를 들어, 상품 리뷰 작성자(유저)의 데이터가 잘못되어, user-api에서 `null`을 리턴하는 경우가 있다. 이 경우 전체 response가 `null`이 된다.
   ![](/assets/img/gql-result1.png)

- 한 쿼리에 여러가지 쿼리를 작성하면 네트워크 통신을 줄일 수 있다.
  - 하지만 gateway 왕복 횟수가 한 번 줄어드는것뿐이지, 뒷 단에서의 통신은 똑같기 때문에 그만큼 지연시간이 더 드는 단점이 있으니 필요할때만 사용하는것을 추천한다.
  ![](/assets/img/gql-result2.png)


## 참고
- Apollo 공식 문서
    - <https://www.apollographql.com/docs/federation/federated-types/overview>
- 블로그
    - <https://dev.to/leoloso/executing-multiple-queries-in-a-single-operation-in-graphql-goe>
    - <https://chanhuiseok.github.io/posts/gql-1/>


