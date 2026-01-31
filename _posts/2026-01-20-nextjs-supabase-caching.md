---
layout: single
title: Next.js 캐싱으로 웹 서버 성능 최적화 (+ Supabase)
category: TIL
toc: true
toc_sticky: true
---

# SSR에서 Full Route Cache까지: Next.js 캐싱으로 주보 페이지 최적화하기

교회 웹사이트에는 매주 업로드되는 주보(교회 소식지)를 제공하는 페이지가 있습니다. 초기 구현은 매 요청마다 동일한 DB 조회와 HTML 렌더링을 반복하는 SSR 방식이었습니다. 주보는 일주일간 내용이 거의 변하지 않는데도 매번 같은 작업을 반복해 불필요한 서버 부하가 발생했습니다.

이 글은 주보 페이지에 Next.js의 캐싱, 특히 Full Route Cache를 적용하여 웹 서버 성능을 최적화한 경험을 다룹니다. Supabase SDK에 캐싱 전략을 적용하는 방법, 실행 환경별 Client 분리 그리고 증가한 책임을 분산하기 위한 Service Layer 설계까지 전체 과정을 공유합니다.

> **기술 스택**
>
> - Next.js 16+ (App Router) / Supabase / TypeScript

## 1. 문제 인식: SSR 기반 렌더링의 한계

### 1.1 초기 상황

주보 페이지는 목록 페이지와 상세 페이지로 구성되어 있으며 다음 요구사항을 만족해야 했습니다.

- SEO와 SNS 공유를 위한 초기 요청 시점의 완성된 HTML 제공
- 새 주보 업로드 시 즉시 반영
- 모바일 환경에서도 빠른 초기 로딩 속도

이를 위해 초기에는 SSR(Server-Side Rendering)을 선택했습니다. 모든 요청마다 서버에서 데이터를 조회하고 HTML을 생성해 반환하는 방식입니다.

하지만 주보는 매주 1회 업로드되고 게시 이후에는 내용이 거의 변경되지 않습니다. 그럼에도 매 요청마다 동일한 DB 조회와 렌더링이 반복되면서 불필요한 서버 연산이 계속 발생하고 있었습니다.

### 1.2 최적화 전략

#### 주보 목록 페이지: Data Cache 적용

목록 페이지는 연도 필터링과 페이지네이션을 위해 쿼리 파라미터(`?year=2026&page=2`)를 사용합니다. 서버 컴포넌트에서 `searchParams`를 참조하면 런타임에 값이 결정되는 Dynamic API 특성상 해당 페이지는 자동으로 동적 렌더링됩니다.

```ts
export default async function BulletinPage({ searchParams }: Props) {
  const { year, page } = await searchParams;
  // ... 
}
```

정적 페이지로 전환하려면 `/bulletin/year/2026/page/2`처럼 경로 파라미터로 변경하고  `generateStaticParams`로 미리 생성할 경로를 정의해야 합니다. 

하지만 목록 페이지는 검색이나 정렬 등 필터가 확장될 수 있어 이를 정적으로 생성할 경우 빌드 시점에 생성해야 할 페이지 수가 빠르게 증가합니다.

따라서 목록 페이지는 동적 렌더링을 유지한 채 데이터 조회 결과에만 캐싱을 적용하는 전략을 선택했습니다.

#### 주보 상세 페이지: Full Route Cache 적용

주보 상세 페이지의 특성은 다음과 같습니다.

- 주보는 매주 1회 업로드되어 페이지 수가 많지 않습니다.
- 게시 이후 내용 변경이 거의 없습니다.
- 외부 공유로 특정 상세 페이지에 대한 접근이 집중됩니다.

이러한 특성상 매 요청마다 HTML을 생성하기보다 정적으로 생성된 렌더링 결과를 재사용하는 편이 더 효과적이었습니다. 

이에 따라 상세 페이지에는 Full Route Cache를 적용하기로 했습니다.

|                       주보 상세 페이지                       |                       주보 목록 페이지                       |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| ![image-20260121190945365](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260121191012812.png) | ![image-20260121191012812](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260121190945365.png) |

### 1.3 Next.js 캐싱: Data Cache와 Full Route Cache

앞에서 선택한 Data Cache와 Full Route Cache는 캐싱 대상이 다릅니다.

Data Cache는 `fetch` 요청의 응답을 캐싱합니다. 페이지가 동적으로 렌더링되더라도 내부에서 동일한 데이터 요청이 반복되면 네트워크 요청 대신 캐시된 응답을 재사용합니다.

Full Route Cache는 페이지의 렌더링 결과 자체를 캐싱합니다. 빌드 시점에 생성된 HTML과 RSC Payload를 서버에 저장해두고 이후 요청에서는 DB 조회나 렌더링 없이 캐시된 결과를 즉시 반환합니다.

Full Route Cache를 적용하려면 페이지가 정적으로 렌더링되어야 하며 이를 위해 다음 조건을 만족해야 합니다.

1. Dynamic Routes의 경우 `generateStaticParams`로 정적 경로 정의
2. `cookies()`, `headers()`, `searchParams` 같은 Dynamic API 미사용
3. 모든 데이터 페칭에 캐싱 적용

특히 세 번째 조건이 중요했습니다. 데이터 페칭이 캐싱되지 않으면 빌드마다 다른 응답이 생성되어 페이지는 동적으로 동작하게 됩니다.

문제는 주보 페이지의 데이터 조회가 Supabase SDK를 통해 이루어지고 있었다는 점이었습니다.

따라서 주보 상세 페이지에 Full Route Cache를 적용하려면 먼저 Supabase SDK를 사용하는 데이터 조회 과정에 Next.js 캐싱을 어떻게 적용할 수 있을지 해결해야 했습니다.



## 2. Supabase SDK에 Next.js 캐싱 적용하기

### 2.1 Supabase SDK와 Next.js 캐싱

Supabase SDK는 내부적으로 `fetch`를 사용합니다. Next.js 환경에서 `fetch`는 기본적으로 `cache: 'auto'` 동작을 하는데 프로덕션 빌드의 정적 페이지에서는 자동으로 캐싱됩니다.

Next.js는 `fetch` 요청에 대해 다음과 같은 방식으로 캐싱 전략을 제어할 수 있습니다.

- `cache` 옵션으로 캐싱 여부 결정
- `next.revalidate`로 갱신 주기 설정
- `next.tags`로 특정 캐시만 선택적으로 무효화

문제는 Supabase SDK가 이러한 Next.js 캐시 옵션을 직접 전달받을 수 있는 인터페이스를 제공하지 않는다는 점이었습니다.

### 2.2 해결 방법 탐색

캐싱 전략을 적용하기 위한 세 가지 대안을 검토했습니다.

#### 방법 1: Supabase REST API 직접 호출

Supabase는 PostgREST 기반 REST API를 제공합니다. `fetch`를 직접 사용하면 Next.js 캐시 옵션을 자유롭게 설정할 수 있습니다.

```ts
await fetch(
  `${NEXT_PUBLIC_SUPABASE_URL}/rest/v1/bulletins?year=eq.2026&order=date.desc&limit=10`,
  {
    next: { tags: ['bulletin'], revalidate: 86400 },
  }
);
```

하지만 Supabase SDK가 제공하는 타입 안전성, 쿼리 빌더, RLS 통합 등 편의 기능을 모두 포기해야 했습니다.

#### 방법 2: unstable_cache 사용

Next.js의 `unstable_cache`로 함수 결과를 캐싱할 수 있습니다.

```ts
import { unstable_cache } from 'next/cache';

const getCachedBulletin = unstable_cache(
  async (id) => {
    const { data } = await supabase
      .from('bulletins')
      .select('*')
      .eq('id', id);
    return data;
  },
  ['bulletin'],
  { revalidate: 86400 }
);
```

모든 쿼리 함수를 래핑해야 하고 캐시 hit/miss를 네트워크 레벨에서 확인할 수 없어 디버깅이 어려웠습니다.

#### 방법 3: Custom Fetch 주입 (채택)

Supabase Client 생성 시 `global.fetch`를 설정하여 커스텀 fetch를 주입하는 방법입니다. SDK의 모든 요청에 Next.js 캐시 옵션을 투명하게 적용할 수 있고 기존 코드 변경이 최소화됩니다.

대신 Supabase Client 생성 시점에 캐시 옵션이 결정되기 때문에 Client 생성 로직이 복잡해질 수 있습니다.

> 이러한 단점은 후에 Service Layer 도입으로 해결하게 됩니다.

결과적으로 Supabase SDK의 타입 안전성과 쿼리 빌더를 그대로 유지하면서 Next.js 캐싱 전략을 적용할 수 있는 방법은 Custom Fetch를 주입하는 방식이었습니다.

```ts
// lib/supabase/custom-fetch.ts - Next.js 캐시 옵션을 받아 fetch를 래핑하는 함수
type NextCacheOptions = { cache?: RequestCache; revalidate?: number; tags?: string[]; };

export const createFetch = ({ cache, tags, revalidate }: NextCacheOptions) => 
  (url: RequestInfo | URL, init?: RequestInit) =>
    fetch(url, {
      ...init,
      ...(cache && { cache }),
      ...(tags || revalidate ? { next: { tags: tags ?? [], revalidate: revalidate ?? 0 } } : {}),
    });
```

**Supabase Client 생성 시 Custom Fetch 주입**

```ts
export const createServerSideClient = async (options = {}) => {
  const cookieStore = await cookies();

  return createServerClient(..., {
    global: { fetch: createFetch({ cache: 'no-store', ...options }) },
    cookies: { /* getAll / setAll 구현 */ }
  });
};
```

기본적으로 `no-store`를 사용하지만 옵션으로 캐시 전략을 덮어쓸 수 있습니다. 이제 주보 상세 조회 시 캐시 옵션을 전달할 수 있습니다.

```ts
export const fetchBulletinDetailById = async (id: string) => {
  const supabase = await createServerSideClient({
    cache: 'force-cache',
    revalidate: 86400,
    tags: ['bulletin', `bulletin-${id}`]
  });
  
  const { data } = await supabase
    .from('bulletins')
    .select(`*, profiles:profiles ( user_name )`)
    .eq('id', id)
    .single();
    
  return { data };
};
```

이 방식으로 Supabase SDK를 사용하는 데이터 조회에도 Next.js 캐싱을 적용할 수 있게 되었습니다. 목록 페이지와 상세 페이지 모두에서 데이터 요청 단위의 캐싱은 정상적으로 동작했습니다.

하지만 Full Route Cache를 적용하려는 과정에서 또 다른 문제가 드러났습니다. 빌드 시점에 실행되는 코드와 요청 시점에 실행되는 코드가 서로 다른 실행 환경을 가지는데 동일한 Supabase Client를 그대로 사용할 수 없었던 것입니다.

<br/>

## 3. Full Route Cache 적용 시도

### 3.1 generateStaticParams 적용

주보 상세 페이지는 Dynamic Route(`/bulletin/[id]`)입니다. 이런 동적 경로를 정적으로 생성하려면 빌드 시점에 어떤 경로들을 미리 생성할지 Next.js에게 알려줘야 합니다. 이때 사용하는 것이 `generateStaticParams`입니다.

빌드 시점에 전체 주보 ID 목록을 조회하고 이를 기반으로 정적 경로를 생성하도록 구현했습니다.

```ts
export async function getAllBulletinIds() {
  const supabase = await createServerSideClient();
  const { data } = await supabase
    .from('bulletins')
    .select('id')
    .order('date', { ascending: false });
  return { data };
}

export async function generateStaticParams() {
  const { data: allBulletins } = await getAllBulletinIds();
  return allBulletins.map((bulletin) => ({
    id: bulletin.id.toString(),
  }));
}
```

하지만 빌드를 실행하자 다음과 같은 에러가 발생했습니다.

```bash
Error: `cookies` was called outside a request scope
Failed to collect page data for /news/bulletin/[id]
```

### 3.2 원인: 빌드 타임과 런타임의 실행 환경 차이

에러 메시지를 따라가며 원인을 확인해보니 `createServerSideClient()` 내부에서 `cookies()`를 호출하고 있었습니다.

```ts
export const createServerSideClient = async (options = {}) => {
  const cookieStore = await cookies(); // ❌ 빌드 타임에서는 실행 불가
  
  return createServerClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
    cookies: {
      getAll() { return cookieStore.getAll(); },
      setAll(cookiesToSet) { /* ... */ },
    },
  });
};
```

`createServerSideClient`는 요청 시점에 실행되는 서버 환경을 전제로 만들어진 Client입니다. 쿠키를 통해 사용자를 식별하고 Supabase의 RLS 정책을 적용하기 위해서입니다.

하지만 `generateStaticParams`는 빌드 타임에 실행됩니다. 아직 사용자 요청이 없어 쿠키가 존재하지 않고 모든 사용자를 위한 정적 HTML을 생성하는 시점이므로 사용자별 인증도 필요 없습니다.

런타임을 전제로 한 Supabase Client를 빌드 타임에 사용하려 했던 것이 문제였습니다. 이 문제를 해결하려면 실행 시점에 맞는 Supabase Client를 분리해서 사용해야 했습니다.

<br/>

## 4. 실행 환경별 Supabase Client 분리

앞선 에러의 원인은 캐시 설정이 아니라 실행 환경에 맞지 않는 Client를 사용한 것이었습니다. Supabase Client는 실행 시점에 따라 요구사항이 달랐습니다.

- 런타임에서는 사용자 요청을 처리하므로 쿠키로 사용자를 식별하고 RLS를 적용해야 합니다. 이때 응답은 사용자마다 달라질 수 있으므로 기본적으로 캐싱하지 않습니다.
- 빌드 타임에서는 정적 HTML을 생성하므로 요청과 쿠키가 존재하지 않고 모든 사용자에게 동일한 공개 데이터만 필요합니다. 이 경우 캐싱을 적극적으로 활용할 수 있습니다.

이 차이를 명확히 분리하기 위해 Supabase Client를 실행 환경 기준으로 나누었습니다.

### 4.1 Server Client와 Static Client 

Server Client는 요청 시점에 실행되는 런타임 환경을 전제로 설계했습니다.

```ts
// lib/supabase/server-client.ts
export const createServerSideClient = async (options = {}) => {
  const cookieStore = await cookies();
  
  return createServerClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
    global: { 
      fetch: createFetch({ cache: 'no-store', ...options }) 
    },
    cookies: {
      getAll() { return cookieStore.getAll(); },
      setAll(cookiesToSet) {
        cookiesToSet.forEach(({ name, value, options }) =>
          cookieStore.set(name, value, options)
        );
      },
    },
  });
};
```

- 쿠키를 통해 사용자를 식별하고 RLS를 적용합니다.
- 기본 캐시 정책은 `no-store`이며 공개 데이터에 한해 옵션으로 캐싱을 허용합니다.



**Static Client**는 빌드 타임과 정적 렌더링을 위한 Client입니다.

```ts
// lib/supabase/static-client.ts
export const createStaticClient = (options = {}) => {
  return createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
    global: { 
      fetch: createFetch({ cache: 'force-cache', ...options }) 
    },
    auth: { 
      persistSession: false,
      autoRefreshToken: false,
      detectSessionInUrl: false
    }
  });
};
```

- 쿠키와 세션 관리를 완전히 비활성화합니다.
- 기본 캐시 정책은 `force-cache`입니다.
- `generateStaticParams`와 정적 페이지 렌더링에 사용합니다.

### 4.2 빌드 타임 에러 해결

빌드 시점에서 Server Client 대신 Static Client를 사용하도록 수정하자 문제가 해결되었습니다.

```tsx
export async function getAllBulletinIds() {
  const supabase = createStaticClient(); // ✅ 빌드 타임에서 안전
  const { data } = await supabase.from('bulletins').select('id');
  return { data };
}
```

이제 `generateStaticParams`가 정상적으로 실행되며 주보 상세 페이지를 정적으로 생성할 수 있게 되었습니다.

<br/>

## 5. Service Layer로 책임 분산

Client를 실행 환경별로 분리하면서 빌드 타임 문제는 해결됐지만 새로운 문제가 드러났습니다. Client가 처리해야 할 책임이 지나치게 많아진 것입니다.

### 5.1 Client 책임 증가

Custom Fetch 주입으로 캐시 정책 결정 책임이 생겼고 환경별 분리로 실행 환경 판단 책임까지 추가되었습니다.

```ts
const supabase = await createServerSideClient({
  cache: 'force-cache',
  revalidate: 86400,
  tags: ['bulletin', `bulletin-${id}`]
});

const { data } = await supabase
  .from('bulletins')
  .select(`*, profiles:profiles ( user_name )`)
  .eq('id', id)
  .single();
```

Client 생성과 쿼리 로직이 강하게 결합되어 있습니다. 동일한 주보 상세 조회 쿼리를 다른 환경에서 사용하려면 Client만 바꿔서 전체를 복사해야 했습니다.

```ts
// 빌드 타임
export const fetchBulletinDetailStatic = (id: string) => {
  const supabase = createStaticClient({ tags: [`bulletin-${id}`] });
  return supabase.from('bulletins')...;
};

// 런타임
export const fetchBulletinDetailServer = async (id: string) => {
  const supabase = await createServerSideClient();
  return supabase.from('bulletins')...; // 동일한 쿼리
};

```

문제의 핵심은 쿼리 로직과 캐시 옵션 전달, 실행 환경이 한 함수에 섞여 있다는 점이었습니다.

### 5.2 관심사 분리를 위한 3-Layer 구조로 분리

이 문제를 해결하기 위해 세 가지 관심사를 분리했습니다. 

- 어떤 데이터를 조회하는가 → 쿼리 로직
- 캐시를 어떻게 적용하는가 → 캐시 정책
- 어디서 실행되는가 → 실행 환경

이를 기준으로 구조를 세 개의 레이어로 나눴습니다.

**Layer 1: 캐시 정책**

```ts
// services/bulletin/bulletin-cache.ts
export const bulletinCache = {
  detail: (id: string) => ({
    tags: ['bulletin', `bulletin-${id}`],
    revalidate: 86400
  }),
  list: () => ({
    tags: ['bulletin', 'bulletin-list'],
    revalidate: 86400
  }),
};
```

캐시 정책만 담당하며 변경 시 한 곳만 수정하면 됩니다.

**Layer 2: 쿼리 로직**

```ts
// services/bulletin/bulletin-service.ts
export const bulletinService = (supabase: SupabaseClient) => ({
  fetchBulletinDetailById: async (id: string) => {
    return supabase
      .from('bulletins')
      .select(`*, profiles:profiles ( user_name )`)
      .eq('id', id)
      .single();
  },

  fetchBulletinList: async ({ year, page = 1 }) => {
    let query = supabase
      .from('bulletins')
      .select('*')
      .order('date', { ascending: false });

    if (year) {
      query = query
        .gte('date', `${year}-01-01`)
        .lte('date', `${year}-12-31`);
    }

    return query.range((page - 1) * 10, page * 10 - 1);
  },
});
```

Service는 Client의 종류나 캐시 여부를 전혀 알지 않습니다. 주어진 Client로 쿼리만 수행합니다.

#### Layer 3: 실행 환경 조합

```ts
// services/bulletin/bulletin-interface.ts
export const fetchBulletinDetailById = (id: string) => {
  const supabase = createStaticClient(bulletinCache.detail(id));
  return bulletinService(supabase).fetchBulletinDetailById(id);
};

export const fetchBulletinDetailServer = async (id: string) => {
  const supabase = await createServerSideClient();
  return bulletinService(supabase).fetchBulletinDetailById(id);
};

export const fetchBulletinList = async (params = {}) => {
  const supabase = await createServerSideClient(bulletinCache.list());
  return bulletinService(supabase).fetchBulletinList(params);
};
```

Client와 캐시 정책을 조합하여 환경별 인터페이스를 제공합니다. 쿼리 로직은 재사용합니다.

### 5.3 책임 분리의 효과

Service Layer를 도입하면서

- 쿼리 로직 중복이 제거되었고
- 캐시 정책 변경이 한 곳으로 모였으며
- 실행 환경에 따른 분기 또한 명확해졌습니다.

레이어가 하나 늘어나 구조는 복잡해졌지만 대신 각 책임의 경계가 분명해지고 변경 영향 범위를 예측할 수 있게 되었습니다.

<br/>

## 6. Full Route Cache 적용 완료

모든 준비가 완료되어 최종적으로 Full Route Cache를 적용했습니다.

### 6.1 페이지 구현

```tsx
// app/news/bulletin/[id]/page.tsx
import { 
  fetchAllBulletinIds, 
  fetchBulletinDetailById, 
  fetchNavigationBulletins 
} from '@/services/bulletin';

export async function generateStaticParams() {
  const { data: allBulletins } = await fetchAllBulletinIds();
  
  // 최근 10개만 빌드 시 생성 (나머지는 첫 요청 시 On-Demand ISR)
  return allBulletins.slice(0, 10).map((bulletin) => ({
    id: bulletin.id.toString(),
  }));
}

export default async function BulletinDetailPage({ params }: Props) {
  const { id } = await params;

  // 병렬 데이터 페칭 (둘 다 Static Client 사용)
  const [bulletinRes, prevNextRes] = await Promise.all([
    fetchBulletinDetailById(id),
    fetchNavigationBulletins(Number(id))
  ]);

  const { data: bulletin } = bulletinRes;
  if (!bulletin) notFound();

  return (
    <MainContainer title="주보">
      <BoardHeader
        title={bulletin.title}
        userName={bulletin.profiles?.user_name ?? '관리자'}
        thumbnail={bulletin.image_url[0]}
      />
      <BoardBody images={bulletin.image_url} />
      <BoardFooter prevNext={prevNextRes.data} />
    </MainContainer>
  );
}
```

두 데이터 페칭 함수 모두 Service Layer를 통해 Static Client를 사용합니다. 이로 인해 빌드 시점에 데이터 조회와 캐싱이 완료되고 런타임에는 데이터 요청이 발생하지 않습니다.

주보 수정 시에는 페이지 경로와 데이터 태그를 함께 무효화해 다음 요청부터 최신 데이터로 다시 생성되도록 했습니다.

```tsx
// app/actions/bulletin.ts
'use server';

import { revalidatePath, revalidateTag } from 'next/cache';

export async function updateBulletin(id: string, data: BulletinData) {
  const supabase = await createServerSideClient();
  
  const { error } = await supabase
    .from('bulletins')
    .update(data)
    .eq('id', id);
    
  if (!error) {
    revalidatePath('/news/bulletin/[id]', 'page');
    revalidateTag(`bulletin-${id}`);
  }
  
  return { error };
}
```

### 6.2 빌드 결과

빌드를 실행하면 다음과 같은 결과를 볼 수 있습니다.

```bash
Route (app)                          
┌ ○ /                                   
├ ● /news/bulletin/[id]                    
│   ├ /news/bulletin/98
│   ├ /news/bulletin/97
│   └ [+8 more paths]
└ λ /news/bulletin                         

○  (Static)   prerendered as static content
●  (SSG)      prerendered as static HTML (uses generateStaticParams)
λ  (Dynamic)  server-rendered on demand
```

`●` 표시는 `generateStaticParams`를 사용하여 정적으로 생성된 페이지를 의미합니다.

**빌드 시 발생하는 일**:

1. `generateStaticParams` 실행 → 전체 주보 ID 조회 (Static Client 사용)
2. 최근 10개 ID를 빌드할 경로로 결정
3. 각 ID에 대해:
   - `fetchBulletinDetailById(id)` 호출 (Static Client + `force-cache`)
   - `fetchNavigationBulletins(id)` 호출 (Static Client + `force-cache`)
   - DB 데이터를 Next.js Data Cache에 저장
   - HTML과 RSC Payload 생성
   - Full Route Cache에 저장
4. 나머지 ID는 첫 요청 시 On-Demand ISR로 생성

**런타임 동작**:

사용자가 `/news/bulletin/98`에 접속하면:

1. Full Route Cache에서 해당 경로 확인
2. 캐시 HIT → 미리 생성된 HTML 즉시 반환
3. DB 조회 없음, 렌더링 없음

### 6.3 성능 측정 결과

**응답 시간 비교**:

| 지표            | 개선 전 (SSR) | 개선 후 (SSG) | 개선율      |
| --------------- | ------------- | ------------- | ----------- |
| 총 응답 시간    | 137.48ms      | 11.96ms       | **91.3% ↓** |
| 서버 응답 대기  | 123.73ms      | 9.08ms        | **92.7% ↓** |
| 콘텐츠 다운로드 | 10.09ms       | 0.66ms        | **93.5% ↓** |

**캐시 헤더 검증**:

```
Cache-Control: s-maxage=86400, stale-while-revalidate=31449600
X-Nextjs-Cache: HIT
X-Nextjs-Prerender: 1
```

- `X-Nextjs-Cache: HIT`: Full Route Cache에서 제공됨
- `X-Nextjs-Prerender: 1`: 빌드 시 생성된 정적 페이지

**최종 성과**:

| 항목        | 개선 전                  | 개선 후                   |
| ----------- | ------------------------ | ------------------------- |
| 응답 시간   | 137.48ms                 | 11.96ms                   |
| DB 쿼리     | 매 요청 실행             | 빌드 시 1회               |
| HTML 렌더링 | 매 요청 실행             | 빌드 시 1회               |
| 서버 부하   | 요청 수에 비례           | 요청 수와 무관            |
| 확장성      | 트래픽 증가 시 부하 증가 | 트래픽 증가해도 부하 일정 |

## 7. 결론

