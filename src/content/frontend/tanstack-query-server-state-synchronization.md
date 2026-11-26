---
title: "TanStack Query로 서버 상태 동기화하기"
description: "useEffect 수동 요청 관리의 한계를 살펴보고 TanStack Query의 캐시, 쿼리 키, staleTime 기반 동기화 방식을 알아본다."
date: "2026-11-26"
tags:
  - TanStack Query
  - Server State
  - React
  - TypeScript
  - Caching
---

# TanStack Query로 서버 상태 동기화하기

프론트엔드에서 API 데이터를 다룰 때 데이터를 컴포넌트가 소유하는 상태로 생각하기 쉽다. 하지만 서버 데이터는 클라이언트가 직접 관리하는 값이 아니라 서버의 현재 상태를 일정 시점에 반영한 정보이다.

TanStack Query는 서버 데이터를 `useState`에 옮겨 담는 방식 대신, 캐시와 서버 데이터를 동기화하는 역할을 맡는다. 이를 통해 중복 요청, 로딩 상태, 에러 상태, 캐시 갱신을 선언적으로 관리할 수 있다.

## 1. 서버 상태와 클라이언트 상태의 차이

클라이언트 상태는 모달 열림 여부, 탭 선택 상태처럼 브라우저 안에서만 관리하는 값이다. 반면 서버 상태는 사용자 정보, 게시글 목록, 거래 내역처럼 서버를 원본으로 두는 값이다.

```text
클라이언트 상태
브라우저가 직접 소유하고 변경

서버 상태
서버가 원본을 소유하고 클라이언트는 조회와 동기화를 담당
```

서버 상태는 여러 컴포넌트에서 공유될 수 있고, 시간이 지나면 오래된 값이 될 수 있으며, 요청 실패와 재시도도 고려해야 한다. TanStack Query는 이런 특성을 쿼리 캐시 안에서 관리한다.

## 2. 설치와 QueryClientProvider 설정

먼저 패키지를 설치한다.

```bash
npm install @tanstack/react-query
```

`QueryClient`는 모든 쿼리의 캐시와 기본 동작을 관리한다. 컴포넌트 바깥에서 한 번만 생성하고, `QueryClientProvider`로 앱 전체에 제공한다.

```tsx
// src/main.tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import App from './App'

const queryClient = new QueryClient()

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  </StrictMode>,
)
```

`QueryClient`를 컴포넌트 안에서 만들면 렌더링마다 캐시가 새로 생성될 수 있다. 따라서 앱 진입점 또는 컴포넌트 바깥에서 생성하는 방식이 안전하다.

## 3. useEffect로 요청을 수동 관리할 때의 한계

`useEffect`로 서버 데이터를 가져오려면 데이터, 로딩, 에러 상태를 각각 만들고 요청 취소나 오래된 응답 처리까지 직접 관리해야 한다.

```tsx
import { useEffect, useState } from 'react'
import { fetchUserData } from '../api/mockApi'
import type { UserData } from '../api/mockApi'

export default function OldUserProfile({ userId }: { userId: number }) {
  const [data, setData] = useState<UserData | null>(null)
  const [isLoading, setIsLoading] = useState(true)
  const [error, setError] = useState<Error | null>(null)

  useEffect(() => {
    let isCancelled = false

    setIsLoading(true)

    fetchUserData(userId)
      .then((response) => {
        if (!isCancelled) {
          setData(response)
          setError(null)
        }
      })
      .catch((error: Error) => {
        if (!isCancelled) setError(error)
      })
      .finally(() => {
        if (!isCancelled) setIsLoading(false)
      })

    return () => {
      isCancelled = true
    }
  }, [userId])

  if (isLoading) return <p>불러오는 중</p>
  if (error) return <p>{error.message}</p>

  return <p>{data?.name}</p>
}
```

이 방식에서는 요청마다 세 종류의 상태와 `isCancelled` 같은 방어 로직이 반복된다. 같은 `userId`를 사용하는 컴포넌트가 여러 개라면 요청과 캐시를 공유하기도 어렵다.

## 4. useQuery로 서버 데이터 동기화하기

`useQuery`는 쿼리 키로 데이터를 구분하고, 요청 함수와 캐시 정책을 함께 선언한다.

```tsx
// src/components/UserProfile.tsx
import { useQuery } from '@tanstack/react-query'
import { fetchUserData } from '../api/mockApi'
import type { UserData } from '../api/mockApi'

export default function UserProfile({ userId }: { userId: number }) {
  const { data, error, isFetching, isPending } = useQuery<UserData, Error>({
    queryKey: ['user', userId],
    queryFn: () => fetchUserData(userId),
    staleTime: 1000 * 60 * 5,
  })

  if (isPending) return <p>서버와 데이터를 동기화하는 중</p>
  if (error) return <p>에러 발생: {error.message}</p>

  return (
    <section>
      <h2>사용자 정보</h2>
      <p>이름: {data.name}</p>
      {isFetching && <span>최신 데이터를 확인하는 중</span>}
    </section>
  )
}
```

`isPending`은 아직 보여줄 캐시 데이터가 없는 초기 상태를 나타낸다. `isFetching`은 초기 요청뿐 아니라 캐시 데이터를 보여주는 동안의 백그라운드 재요청도 포함한다. 따라서 캐시된 데이터를 먼저 보여주면서 최신 데이터를 다시 가져오는 상태도 표현할 수 있다.

## 5. queryKey와 캐시

`queryKey`는 데이터를 식별하는 주소이다. 요청 함수에서 사용하는 값은 쿼리 키에도 포함해야 한다.

```tsx
useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUserData(userId),
})
```

`userId`가 바뀌면 쿼리 키도 달라진다. TanStack Query는 이를 다른 데이터로 인식하고 별도의 캐시를 사용한다. 같은 키를 사용하는 컴포넌트는 진행 중인 요청과 결과를 공유할 수 있으므로 중복 요청을 줄일 수 있다.

## 6. staleTime과 Stale-While-Revalidate

`staleTime`은 캐시 데이터를 신선하다고 판단하는 시간이다.

```tsx
staleTime: 1000 * 60 * 5
```

위 설정에서는 데이터를 5분 동안 신선한 상태로 취급한다. 이 시간 안에 같은 쿼리를 다시 사용하면 캐시 데이터를 우선 사용한다. 시간이 지나 데이터가 오래된 상태가 되면, 설정과 상황에 따라 다시 요청해 최신 값으로 갱신한다.

```text
캐시 데이터 표시
    ↓
백그라운드 요청
    ↓
최신 응답 수신
    ↓
화면 갱신
```

기본 `staleTime`은 `0`이므로, 특별한 설정이 없으면 캐시 데이터는 바로 오래된 상태로 취급된다. 모든 데이터에 5분을 적용하기보다 사용자 정보, 거래 내역, 실시간 가격처럼 데이터 성격에 맞게 시간을 결정해야 한다.

## 7. 요청 취소와 경쟁 상태

TanStack Query는 쿼리의 생명 주기와 상태를 관리한다. 다만 네트워크 요청 자체를 취소하려면 `queryFn`이 전달받는 `signal`을 `fetch`에 연결해야 한다.

```tsx
async function fetchUserData(userId: number, signal: AbortSignal) {
  const response = await fetch(`/api/users/${userId}`, { signal })

  if (!response.ok) {
    throw new Error('사용자 정보를 가져오지 못했습니다.')
  }

  return response.json()
}

useQuery({
  queryKey: ['user', userId],
  queryFn: ({ signal }) => fetchUserData(userId, signal),
})
```

이렇게 작성하면 더 이상 필요하지 않은 쿼리가 취소될 때 브라우저 요청에도 취소 신호를 전달할 수 있다. 요청 취소를 지원하지 않는 함수라면 TanStack Query가 상태 업데이트를 관리하더라도 네트워크 요청 자체는 계속 진행될 수 있다.

## 8. 테스트로 확인할 흐름

### 중복 요청 제거

같은 `userId`를 가진 `UserProfile`을 두 개 배치한다.

```tsx
<UserProfile userId={1} />
<UserProfile userId={1} />
```

브라우저 Network 탭에서 같은 쿼리 키에 대한 요청이 불필요하게 여러 번 발생하지 않는지 확인한다.

### 캐시와 재검증

데이터를 한 번 불러온 뒤 컴포넌트를 다시 마운트한다. 캐시 데이터가 있다면 즉시 기존 데이터를 표시하고, 데이터가 오래된 상태라면 백그라운드에서 최신 값을 다시 확인하는지 검증한다.

### 타입 안정성

`data.name`을 `data.nane`처럼 잘못 작성하면 TypeScript가 에디터에서 오류를 표시하는지 확인한다. `useQuery<UserData, Error>` 제네릭은 응답 데이터와 오류 타입을 명확하게 전달한다.

## 정리

- 서버 데이터는 컴포넌트가 소유하는 값이 아니라 서버 상태를 반영하는 정보이다.
- `QueryClientProvider`는 앱 전체가 같은 쿼리 캐시를 공유하도록 만든다.
- `queryKey`는 데이터를 식별하며 요청 함수에 사용하는 값도 포함해야 한다.
- `staleTime`은 캐시 데이터를 신선하게 취급할 시간을 결정한다.
- `isPending`은 초기 데이터 없음 상태를, `isFetching`은 백그라운드 갱신을 포함한 요청 상태를 나타낸다.
- 요청 취소가 필요하면 `queryFn`의 `signal`을 실제 요청 함수에 연결한다.

## 참고 자료

- [TanStack Query useQuery 공식 문서](https://tanstack.com/query/latest/docs/framework/react/reference/functions/useQuery)
- [TanStack Query Important Defaults](https://tanstack.com/query/latest/docs/framework/react/guides/important-defaults)
- [TanStack Query Query Keys](https://tanstack.com/query/latest/docs/framework/react/guides/query-keys)
