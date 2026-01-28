---
layout: single
title: Next.js와 Supabase 캐싱으로 웹 서버 성능 최적화
category: TIL
toc: true
toc_sticky: true
---

# SSR 페이지 캐싱 과정에서 실행 환경별 데이터 접근 책임을 분리하게 된 이유

교회 웹사이트의 주보 페이지는 주 1회 업로드되고 이후 수정은 거의 없지만 새로운 주보가 올라오면 즉시 반영되어야 합니다. 또한 공유가 빈번하여 SEO와 응답 속도가 중요한 페이지입니다.

초기에는 항상 최신 데이터를 제공하고자 요청 시마다 서버에서 HTML을 생성하는 SSR 방식을 적용했습니다. 하지만 주보는 업로드 후 다음 주보가 올라오기 전까지 내용이 거의 바뀌지 않는데도 매 요청마다 동일한 조회와 렌더링이 반복되는 비효율이 발생했습니다.

이 글에서는 이러한 문제를 해결하기 위해 Next.js 캐시를 Supabase SDK 요청에 적용한 과정을 다룹니다. 단순히 캐시만 적용하는 것이 아니라 실행 환경별 Client 분리와 Service Layer 도입이 필요했던 이유와 그 구현 방법까지 상세히 살펴보겠습니다.

사용한 기술들은 다음과 같습니다.

- **Framework:** Next.js 16+ (App Router)
- **Database:** Supabase
 - **Language:** TypeScript

<br/>

## 1. 주보 페이지 최적화 전략

### 1.1 요구사항 분석

주보 페이지는 다음 세 가지 요구사항을 동시에 충족해야 했습니다.

- 검색 노출(SEO)과 SNS 공유를 위해 서버에서 완성된 HTML과 메타데이터를 제공
- 새 주보가 업로드되면 실시간으로 반영
- 특정 시간대 모바일 접속 급증 시에도 빠른 응답 속도 보장

이 세 가지를 동시에 만족하기 위해 처음 선택한 방식은 SSR(Server-Side Rendering)이었습니다. 사용자가 접속하는 순간 서버가 페이지를 생성하므로 완성된 HTML을 제공하면서도 데이터의 최신성을 보장할 수 있었기 때문입니다.

### 1.2 SSR의 한계와 개선 방향

하지만 주보 데이터는 일주일간 변하지 않는 정적인 특성을 가집니다. 데이터 변화가 없음에도 매 접속마다 DB 조회와 렌더링을 반복하는 SSR 방식은 서버 자원을 불필요하게 소모했습니다.

이를 해결하기 위해 미리 생성된 결과물을 재사용하는 방식으로의 전환을 고려했습니다. 하지만 주보 페이지는 목록 페이지와 상세 페이지로 나뉘며 각각 다른 특성을 가지고 있었습니다.

|                       주보 목록 페이지                       |                       주보 상세 페이지                       |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| ![image-20260121191012812](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260121190945365.png) | ![image-20260121190945365](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260121191012812.png) |

### 1.3 페이지별 최적화 전략

#### 주보 목록 페이지: 동적 렌더링 + 데이터 캐싱

목록 페이지는 연도 필터링과 페이지네이션 처리를 위해 쿼리 파라미터(`?year=2026&page=2`)를 사용합니다. 서버 컴포넌트에서 `searchParams`를 참조하면 런타임에 값이 결정되는 Dynamic API 특성상 해당 페이지는 자동으로 동적 렌더링됩니다.

```tsx
export default async function BulletinPage({ searchParams }: Props) {
  const { year, page } = await searchParams;
  // ... 
}
```

정적 페이지로 전환하려면 `/bulletin/year/2026/page/2`와 같이 경로 파라미터로 변경하고 `generateStaticParams`로 모든 조합을 미리 생성해야 합니다. 하지만 이는 빌드 시간과 관리 비용 측면에서 비효율적이라 판단했습니다.

따라서 목록 페이지는 동적 렌더링을 유지하되 데이터 조회 결과만 캐싱하는 방식을 선택했습니다.

#### 주보 상세 페이지: 정적 생성 + Full Route Cache

반면 상세 페이지는 게시 후 수정이 거의 없고 외부 공유가 빈번한 특성상 SEO 최적화와 빠른 응답이 중요했습니다.

이러한 이유로 접속 시점이 아닌 빌드 시점에 미리 HTML을 생성하는 정적 생성(Static Generation) 방식을 채택했습니다. 이렇게 생성된 페이지는 서버의 Full Route Cache에 보관되어 요청 시 추가 연산 없이 캐시된 결과물을 즉시 반환합니다.

![Diagram showing the default caching behavior in Next.js for the four mechanisms, with HIT, MISS and SET at build time and when a route is first visited.](https://nextjs.org/_next/image?url=https%3A%2F%2Fh8DxKfmAPhn8O0p3.public.blob.vercel-storage.com%2Fdocs%2Fdark%2Fcaching-overview.png&w=3840&q=75)



## 2. Supabase SDK에 Next.js 캐싱 적용하기

### 2.1 문제: Supabase SDK는 캐시 옵션을 지원하지 않음

Next.js의 캐싱 기능(`cache`, `revalidate`, `tags`)은 기본적으로 `fetch` API를 통해 동작합니다. 하지만 Supabase SDK는 내부적으로 HTTP 요청을 처리하면서도 캐시 옵션을 직접 전달할 수 있는 인터페이스를 제공하지 않습니다.

```ts
const { data } = await supabase
  .from('bulletins')
  .select('*')
  .order('date', { ascending: false });
```

### 2.2 해결 방법 검토

#### 방법 1: Supabase REST API를 fetch로 직접 호출

```ts
await fetch(
  `${NEXT_PUBLIC_SUPABASE_URL}/rest/v1/bulletins?year=eq.2026&order=date.desc&limit=10`,
  {
    next: { tags: ['bulletin'], revalidate: 86400 },
  }
);
```

- 장점: Next.js 캐시 옵션을 직접 제어 가능
- 단점: Supabase SDK의 편리한 쿼리 빌더 대신 URL 쿼리 문자열을 직접 관리해야 함

#### 방법 2: unstable_cache 사용

```ts
const getBulletins = unstable_cache(
  async () => {
    return supabase
      .from('bulletins')
      .select('*')
      .order('date', { ascending: false });
  },
  ['bulletins'],
  { revalidate: 86400 }
);
```

- 장점: Supabase SDK를 그대로 사용 가능
- 단점: 캐시 hit/miss를 네트워크 레벨에서 확인할 수 없어 디버깅이 어려움

#### 방법 3: Custom Fetch를 Supabase Client에 주입 (채택)

```ts
export const createFetch = ({ cache, tags, revalidate }: NextCacheOptions) => 
  (url: RequestInfo | URL, init?: RequestInit) =>
    fetch(url, {
      ...init,
      ...(cache && { cache }),
      ...(tags || revalidate ? { next: { tags: tags ?? [], revalidate: revalidate ?? 0 } } : {}),
    });
```

- 장점: Supabase SDK의 편리함 유지, Next.js 캐시 적용 가능, 네트워크 레벨에서 캐시 동작 확인 가능
- 단점: Supabase Client 생성 시점에 캐시 옵션이 결정됨, Client 생성 로직이 복잡해질 수 있음

> 이러한 단점은 후에 Service Layer 도입으로 해결하게 됩니다.

### 2.3 Custom Fetch 구현 및 적용

**Supabase Client 생성 시 Custom Fetch 주입**

```ts
export const createServerSideClient = async (options: NextCacheOptions = {}) => {
  return createServerClient<Database>(SUPABASE_URL, SUPABASE_KEY,
    {
      global: {
        fetch: createFetch({ cache: 'no-store', ...options }),
      },
    }
  );
};
```

이 방식을 통해 Supabase SDK의 모든 기능을 사용하면서도 Next.js 캐싱을 제어할 수 있게 되었습니다.

### 2.4 주보 목록 페이지에 캐싱 적용

주보 목록 페이지는 데이터 변경이 적어 `force-cache`와 86,400초(24시간) revalidate를 적용했습니다.

```tsx
// page.tsx
const supabase = await createServerSideClient({
  cache: 'force-cache',
  revalidate: 86400
});


const { data, error } = await getBulletins({ year, page });
```

새 주보가 업로드되면 연도나 페이지 에 관계없이 모든 목록 페이지의 캐시를 갱신해야 했기 때문에 경로 기반 무효화(`revalidatePath`)를 적용했습니다.

```ts
// bulletin.actions.ts
export const createBulletinAction = async (formData: FormData) => {
  // 주보 생성 로직
  revalidatePath('/news/bulletin');
}
```

### 2.5 최적화 성과 (수정중)

- **API 응답 속도**: 112ms → 2ms (약 56배 향상)
- **TTFB**: 104.98ms → 50.53ms (51.9% 개선)
- **API 요청 빈도**: 매 요청 → 하루 1회







|구현 결과|
|:-:|
|![스크린 캡처_20260124_210640](\assets\images\2026-01-20-nextjs-supabase-caching\스크린 캡처_20260124_210640.webp)|

<br/>

## 3. 빌드 타임의 제약: Dynamic API와의 충돌

### 3.1 Full Route Cache 적용 조건

Next.js에서 Dynamic Route에 Full Route Cache를 적용하려면 다음 조건을 만족해야 합니다.

1. `generateStaticParams`를 사용해 빌드 타임에 경로 확정
2. `cookies`, `headers` 같은 Dynamic Function을 사용하지 않음
3. 데이터 캐싱이 적용됨

주보 상세 페이지는 `[id]` 기반 Dynamic Route였고 발행 이후 거의 수정되지 않는 콘텐츠였기 때문에 Full Route Cache를 적용하기에 적합했습니다.

### 3.2 Full Route Cache 적용 시도

주보 상세 페이지를 Full Route Cache로 전환하기 위해 빌드 시점에 생성할 페이지 경로를 확정해야 했습니다. 이를 위해 `generateStaticParams` 함수를 구현했습니다.

```tsx
export async function getAllBulletinIds() {
  const supabase = await createServerSideClient(); // 문제 발생 지점
  const { data, error } = await supabase
    .from(BULLETIN_BUCKET)
    .select('id')
    .order('date', { ascending: false });

  if (error) throw error;
  return { data, error };
}

export async function generateStaticParams() {
  const { data: allBulletins, error } = await getAllBulletinIds();
  
  return allBulletins.map((bulletin) => ({
    id: bulletin.id.toString(),
  }));
}
```
전체 주보 ID 목록을 조회하여 각 ID에 대한 페이지를 빌드 시점에 미리 생성하고자 했으나 빌드 과정에서 에러가 발생했습니다.


```bash
Error: `cookies` was called outside a request scope
Failed to collect page data for /news/bulletin/[id]
```

### 3.3 문제의 원인

문제는 `getAllBulletinIds()` 함수 안에 있었습니다. 이 함수는 `createServerSideClient()`를 호출하고 있었는데, 이 Client는 내부적으로 `cookies()` 함수를 사용해 사용자 세션을 확인합니다.

```ts
// createServerSideClient 내부
const cookieStore = await cookies(); // 여기서 에러 발생!
```

그런데 `generateStaticParams`는 빌드 타임에 실행됩니다. 빌드 단계에서는 아직 사용자 요청이 들어오지 않았기 때문에 쿠키나 세션 같은 요청 컨텍스트가 존재하지 않습니다.

**즉, 실행 시점이 다른 환경에서 잘못된 Client를 사용한 것이 문제였습니다.**

이 문제를 해결하려면 빌드 타임에서도 안전하게 동작하는 별도의 Client가 필요했습니다.

<br/>

## 4. 실행 환경별 Supabase Client 분리

### 4.1 실행 환경별로 요구사항이 다른 이유

근본적인 문제는 Supabase Client가 동작하는 방식이 실행 환경에 따라 완전히 달라야 한다는 것이었습니다.

**브라우저와 서버 런타임 환경**

이 환경에서는 사용자를 식별해야 합니다. 사용자가 로그인했는지, 어떤 권한을 가졌는지 확인하려면 쿠키에 저장된 세션 정보가 필요합니다. Supabase의 RLS는 이 정보를 바탕으로 "이 사용자가 이 데이터에 접근할 수 있는가?"를 판단합니다.

**빌드 타임 환경**

반면 빌드 시점에는 모든 사용자에게 보여줄 정적 HTML을 미리 만드는 과정이므로 특정 사용자의 권한이나 세션을 확인할 필요가 없습니다. 오히려 공개 데이터를 빠르게 조회하는 것이 목표입니다.

**캐싱 전략의 차이**

또한 각 환경에서 요구되는 캐싱 전략도 달랐습니다.

- 런타임: 사용자마다 다른 데이터를 보여줘야 하므로 기본적으로 캐싱하지 않음(`no-store`)
- 빌드 타임: 모두에게 같은 데이터를 보여주므로 적극적으로 캐싱(`force-cache`)

이러한 차이를 인식하고 Supabase Client를 실행 환경별로 분리하기로 결정했습니다.

### 4.2 Client 유형별 비교

| Client 유형    | 실행 환경              | 쿠키/세션 접근 | Next.js 캐시 적용 | 핵심 용도                                       |
| -------------- | ---------------------- | -------------- | ----------------- | ----------------------------------------------- |
| Browser Client | 브라우저               | ○              | ✕                 | 실시간 구독, 사용자 인터랙션(UI)                |
| Server Client  | 서버 런타임(요청 처리) | ○              | △                 | RLS 기반 조회/쓰기, Server Action/Route Handler |
| Static Client  | 빌드 타임 / ISR        | ✕              | ○                 | `generateStaticParams`, 정적 페이지 생성        |
| Admin Client   | 서버 런타임            | ✕              | ✕                 | RLS 우회(관리자/시스템 작업)                    |

### 4.3 Client별 상세 구현과 설계 근거

#### 1) Browser Client - 브라우저 환경

```ts
'use client';
let client: ReturnType<typeof createBrowserClient<Database>> | undefined;

export function getSupabaseBrowserClient() {
  if (client) return client;
  client = createBrowserClient<Database>(...);
  return client;
}
```

- 브라우저 환경에서는 페이지 이동과 리렌더링이 빈번하게 발생합니다. 매번 새 인스턴스를 생성하면 중복된 웹소켓 연결로 리소스가 낭비되므로, 첫 호출 시 한 번만 생성하고 재사용합니다.

- 브라우저 Client는 서버 렌더링에 영향을 주지 않으므로 Next.js 서버 캐싱을 적용하지 않습니다.

#### 2) Server Client - 서버 런타임 (요청 처리)

```tsx
export const createServerSideClient = async (options = {}) => {
  const cookieStore = await cookies();

  return createServerClient(..., {
    global: { fetch: createFetch({ cache: 'no-store', ...options }) },
    cookies: { /* getAll / setAll 구현 */ }
  });
};
```

- 서버 런타임에서는 각 요청마다 다른 사용자가 접속하므로 쿠키를 통해 사용자를 식별합니다.
- Supabase의 RLS는 사용자 정보를 기반으로 데이터 접근 권한을 제어합니다. 
- 사용자별로 다른 데이터를 보여줘야 하므로 기본적으로 캐싱하지 않습니다. 캐싱 시 다른 사용자의 데이터가 잘못 노출될 수 있기 때문입니다.
- 공개 데이터처럼 사용자와 무관한 경우 선택적으로 캐싱할 수 있습니다.

#### 3) Static Client - 빌드 타임/ISR

```tsx
export const createStaticClient = (options: NextCacheOptions = {}) => {
  return createClient<Database>(..., {
    global: { fetch: createFetch({ cache: 'force-cache', ...options }) },
    auth: { persistSession: false, ... }
  });
};
```

- 특정 사용자가 아닌 모든 사용자를 위한 정적 HTML을 생성하므로 세션 관리가 필요 없습니다.
- `force-cache`를 통해 API 응답을 캐싱하여 빌드 속도와 페이지 로드 성능을 극대화합니다.

#### 4) Admin Client - RLS 우회

```tsx
export const createAdminServerClient = (): SupabaseClient<Database> => {
  return createClient<Database>(..., SERVICE_ROLE_KEY, {
    auth: { persistSession: false, ... }
  });
};
```

- `SERVICE_ROLE_KEY`를 사용하여 모든 RLS를 우회하고 데이터베이스 전체에 접근 가능합니다.
- 관리자 대시보드, 배치 작업 등 시스템 레벨에서 모든 데이터에 접근해야 하는 경우에만 사용합니다.
- 키 노출 시 심각한 보안 사고로 이어지므로 반드시 서버 환경에서만 사용해야 합니다.
- 관리자 작업은 항상 최신 데이터를 기반으로 수행되어야 하므로 캐싱하지 않습니다.

### 4.4 Client 분리 후 남은 문제

Client를 환경별로 분리하여 빌드 타임 에러는 해결했습니다. Static Client를 사용하면 `cookies()` 없이도 빌드 시점에 안전하게 데이터를 조회할 수 있었습니다.

```tsx
// generateStaticParams에서 Static Client 사용
export async function getAllBulletinIds() {
  const supabase = createStaticClient(); // cookies() 호출 없음
  const { data, error } = await supabase
    .from('bulletins')
    .select('id');
  
  return { data, error };
}
```

하지만 실제로 여러 페이지에서 사용하다 보니 새로운 문제가 나타났습니다.

동일한 주보 상세 조회 쿼리를 다른 환경에서 사용하려면 Client만 바꿔서 호출해야 하는데, 이 과정에서 쿼리 로직이 중복되기 시작했습니다.

```tsx
// ISR 상세 페이지용
export const fetchBulletinDetailStatic = (id: string) => {
  const supabase = createStaticClient({ 
    tags: [`bulletin-${id}`], 
    revalidate: 86400 
  });
  return supabase.from('bulletins').select('*').eq('id', Number(id)).single();
};

// CSR 수정 페이지용
export const fetchBulletinDetailBrowser = (id: string) => {
  const supabase = getSupabaseBrowserClient();
  return supabase.from('bulletins').select('*').eq('id', Number(id)).single();
};
```

Client를 분리하면서 빌드 타임 오류는 해결했지만 새로운 문제가 나타났습니다.

동일한 주보 상세 조회 쿼리를 다른 환경에서 사용하려면 Client만 바꿔서 호출해야 하는데, 이 과정에서 쿼리 로직이 중복되기 시작했습니다.

이 문제를 해결하기 위해 캐시 정책, 쿼리 로직, 실행 환경을 각각 분리하는 Service Layer를 도입했습니다.

<br/>

## 5. Service Layer 도입: 관심사의 분리

### 5.1 3-Layer 아키텍처 설계

Client 분리로 발생한 중복 문제를 해결하기 위해 데이터 페칭 관련 책임을 세 개의 레이어로 나누었습니다.

```
Interface Layer (환경 조합) 
    ↓
Cache Config (캐시 정책) + Supabase Client (환경별 Client)
    ↓
Service Layer (쿼리 로직)
```

각 레이어가 명확한 단일 책임을 가지도록 설계했습니다. 이제 순서대로 각 레이어를 살펴보겠습니다.

#### Layer 1: 캐시 정책 중앙 관리

먼저 해결해야 할 문제는 캐시 정책이 호출부마다 흩어져 있다는 것이었습니다. 동일한 데이터인데 어떤 곳에서는 `revalidate: 86400`을, 다른 곳에서는 `revalidate: 43200`을 사용하면 일관성이 깨집니다.

이를 해결하기 위해 캐시 정책을 한 곳에서 정의하는 Cache Config를 만들었습니다.

```ts
// services/bulletin/bulletin-cache.ts
const ROOT = 'bulletin';

export const bulletinCache = {
  list: () => ({
    tags: [ROOT, 'bulletin-list'],
    revalidate: 86400  // 24시간
  }),
  detail: (id: string | number) => ({
    tags: [ROOT, 'bulletin-detail', `bulletin-detail-${id}`],
    revalidate: 86400
  }),
} as const;
```

#### Layer 2: 순수한 쿼리 로직 분리

다음 문제는 쿼리 로직이 여러 함수에 중복되어 있다는 것이었습니다. 주보 상세 조회 쿼리를 Static Client용, Browser Client용으로 각각 작성하면 쿼리 변경 시 모든 함수를 수정해야 합니다.

이를 해결하기 위해 Client와 무관하게 순수한 쿼리 로직만 담당하는 Service Layer를 만들었습니다.

Service Layer는 Client를 전달받아 데이터베이스 쿼리만 수행합니다.

```ts
// services/bulletin/bulletin-service.ts
export const bulletinService = (supabase: SupabaseClient<Database>) => ({
  // 주보 목록 조회 
  fetchBulletinList: async ({ year, page = 1, limit = 10 }) => { ... },
  
  // 주보 상세 정보 조회
  fetchBulletinDetailById: async (id: string) => { ... },
  
  // 정적 생성을 위한 전체 ID 목록 조회
  fetchAllBulletinIds: async () => { ... },
});
```

#### Layer 3: 실행 환경 조합

이제 마지막으로 어떤 Client를 사용할지와 어떤 캐시 정책을 적용할지를 결정하는 Interface Layer를 만듭니다. 이 레이어는 페이지에서 실제로 호출할 함수들입니다.

```ts
// services/bulletin/bulletin-interface.ts

// 주보 목록 (Server Client + Cache)
export const fetchBulletinList = (params = {}) => {
  const supabase = await createStaticClient(bulletinCache.list());
  return bulletinService(supabase).fetchBulletinList(params); 
};

// 주보 상세 - 정적 페이지용 (Static Client + Cache)
export const fetchBulletinDetailById = (id: string) => {
  const supabase = createStaticClient(bulletinCache.detail(id));
  return bulletinService(supabase).fetchBulletinDetailById(id);
};

// 주보 상세 - 브라우저용 (Browser Client + No Cache)
export const fetchBulletinDetailBrowser = (id: string) => {
  const supabase = getSupabaseBrowserClient();
  return bulletinService(supabase).fetchBulletinDetailById(id);
};

// 전체 ID - 빌드용 (Static Client)
export const fetchAllBulletinIds = () => {
  const supabase = createStaticClient();
  return bulletinService(supabase).fetchAllBulletinIds();
};
```

### 5.2 Service Layer 도입 결과

Service Layer를 도입한 후 페이지 컴포넌트는 복잡한 Client 생성이나 캐시 설정을 전혀 신경 쓰지 않게 되었습니다.

tsx

```tsx
// app/news/bulletin/[id]/page.tsx
import { fetchBulletinDetailById } from '@/services/bulletin';

export default async function BulletinDetailPage({ params }: Props) {
  const { id } = await params;
  const { data: bulletin, error } = await fetchBulletinDetailById(id);

  return <BulletinDetail bulletin={bulletin} />;
}
```

페이지는 비즈니스 로직에만 집중하고 나머지는 Service Layer가 알아서 처리합니다.

- **Cache Config**: 캐시 정책이 변경되면 수정
- **Service**: 쿼리 로직이 변경되면 수정
- **Interface**: 새로운 실행 환경이 필요하면 추가
- **Page**: 아무것도 수정하지 않아도 자동으로 변경사항 반영

각 레이어가 단일 책임을 가지므로 변경의 영향 범위가 명확하고 예측 가능합니다.


## 6. 주보 상세 페이지: Full Route Cache 전환

