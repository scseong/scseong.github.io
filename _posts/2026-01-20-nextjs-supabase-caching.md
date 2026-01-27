---
layout: single
title: 주보 페이지로 설계한 Next.js 데이터 페칭 최적화 및 캐싱 전략
category: TIL
toc: true
toc_sticky: true
---

# 주보 페이지로 설계한 Next.js 데이터 페칭 최적화 및 아키텍처 개선

교회 웹사이트에서 주보는 주 1회 업로드되고 이후 수정은 거의 없지만 업로드되면 바로 반영되어야 합니다. 공유가 잦아 SEO와 응답 속도도 중요했습니다.

초기에는 항상 최신 데이터를 제공하고자 요청 시 서버에서 HTML을 생성하여 응답하는 SSR을 적용했습니다. 하지만 주보는 업로드된 뒤 다음 주보가 올라오기 전까지 내용이 거의 바뀌지 않았고 요청마다 같은 조회와 렌더링이 반복됐습니다.

이 글에서는 서버 최적화를 위해 Supabase SDK 요청에 Next.js 캐시를 적용했습니다. 그 과정에서 실행 환경별 Client 분리와 Service Layer 도입까지 필요했던 이유와 적용 방법을 다룹니다.

> 사용한 기술들은 다음과 같습니다.
>
> - **Framework:** `Next.js 16+ (App Router)`
> - **Database:** `Supabase`
> - **Language:** `TypeScript`



<br/>

## 1. 주보 페이지 최적화: 아키텍처 설계

주보 페이지는 다음과 같은 요구사항을 충족해야 합니다.

- 검색 노출과 SNS 공유를 위해 서버 측에서 완성된 HTML과 메타데이터 제공
- 주 1회 게시되며 새 주보가 업로드되면 즉시 반영되어야 함
- 특정 시간대 모바일 접속이 급증하는 환경에서도 안정적이고 빠른 응답 속도를 보장

위 세 가지 목표를 동시에 충족하기 위해 처음 선택한 것은 실시간 생성 방식(SSR)이었습니다. 사용자가 접속하는 순간 서버가 페이지를 생성하므로 완성된 HTML을 제공하면서도 데이터의 최신성을 보장할 수 있었기 때문입니다.

하지만 주보 데이터는 일주일간 변하지 않는 정적인 특성을 가집니다. 데이터 변화가 없음에도 매 접속마다 DB 조회와 렌더링을 반복하는 SSR 방식은 서버 자원을 불필요하게 소모하는 문제가 있었습니다.

이러한 문제를 해결하고자 미리 생성된 결과물을 재사용하는 정적 렌더링 방식으로의 전환을 고려했습니다. 

|                       주보 목록 페이지                       |                       주보 상세 페이지                       |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| ![image-20260121191012812](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260121190945365.png) | ![image-20260121190945365](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260121191012812.png) |


### 1.1 주보 목록 페이지: 동적 렌더링 유지와 데이터 캐싱

주보 목록 페이지는 연도 필터링과 페이지네이션 처리를 위해 쿼리 파라미터를 사용합니다. 이때 서버 컴포넌트에서 `searchParams`를 참조하면 런타임에 값이 결정되는 Dynamic API 특성상 해당 페이지는 자동으로 동적 렌더링됩니다.

```tsx
export default async function BulletinPage({ searchParams }: Props) {
  const { year, page } = await searchParams;
  // ... 
}
```

주보 목록을 정적 페이지로 전환하기 위해서는 기존의 `/bulletin?year=2026&page=2` 쿼리 파라미터를 `/bulletin/year/2026/page/2` 경로 파라미터로 변경해야 했습니다. 실시간 요청에 의존하는 `searchParams`와 달리 경로 파라미터는 `generateStaticParams`를 통해 빌드 타임에 페이지를 미리 정적으로 생성할 수 있기 때문입니다.

하지만 모든 필터 조합을 정적 페이지로 미리 생성하는 것은 빌드 타임과 관리 비용 면에서 비효율적이라 판단했습니다. 따라서 목록 페이지는 동적 렌더링을 유지하되 데이터 조회 결과만 캐싱하는 방식을 적용했습니다.

### 1.2 주보 상세 페이지: Full Route Cache를 통한 성능 극대화

반면 상세 페이지는 발행 후 수정이 거의 없고 외부 공유가 빈번한 특성상 SEO 최적화와 즉각적인 응답 속도가 무엇보다 중요했습니다.

이러한 이유로 접속 시점에 페이지를 만드는 대신 빌드 시점에 미리 HTML을 생성하는 정적 생성(Static Generation) 방식을 채택했습니다. 이렇게 생성된 페이지는 서버의 Full Route Cache에 보관되어 요청 시 추가 연산 없이 캐시된 결과물을 즉시 반환하므로 사용자에게 빠른 응답 속도를 제공합니다.

![Diagram showing the default caching behavior in Next.js for the four mechanisms, with HIT, MISS and SET at build time and when a route is first visited.](https://nextjs.org/_next/image?url=https%3A%2F%2Fh8DxKfmAPhn8O0p3.public.blob.vercel-storage.com%2Fdocs%2Fdark%2Fcaching-overview.png&w=3840&q=75)



## 2. Supabase SDK와 Next.js 캐싱 통합

Next.js의 강력한 캐싱 기능(`cache`, `revalidate`, `tags`)은 기본적으로 고유한 `fetch` API를 통해 동작합니다. 하지만 Supabase SDK는 내부적으로 HTTP 요청을 처리하면서도 캐시 옵션을 직접 전달할 수 있는 인터페이스를 제공하지 않아 캐시를 적용할 수 없었습니다.

### 2.1 캐시 적용을 위한 대안 검토

#### 1) Supabase REST API를 fetch로 직접 호출

Supabase SDK를 사용하지 않고 Supabase가 제공하는 PostgREST API를 fetch로 직접 호출하는 방식입니다. 이 경우 Next.js의 fetch를 그대로 사용할 수 있기 때문에 캐시 옵션을 제어할 수 있습니다.

```ts
await fetch(
  `${NEXT_PUBLIC_SUPABASE_URL}/rest/v1/bulletins?year=eq.2026&order=date.desc&limit=10`,
  {
    next: { tags: ['bulletin'], revalidate: 84600 },
  }
);
```

다만 이 방식은 기존에 사용하던 Supabase SDK의 조회 방식 대신 조건/정렬/페이지네이션을 모두 URL 쿼리 문자열로 직접 관리해야 했습니다. 

#### 2) unstable_cache를 사용하는 방식

다음으로 고려한 것은 서버 함수의 실행 결과 자체를 캐싱하는 `unstable_cache`였습니다.

```ts
const getBulletins = unstable_cache(
  async () => {
    return supabase
      .from('bulletins')
      .select('*')
      .order('date', { ascending: false });
  },
  ['bulletins'],
  { revalidate: 84600 }
);
```

이 방식의 장점은 Supabase SDK를 그대로 사용할 수 있다는 점입니다. 많은 변경 없이 캐시를 적용할 수 있고 revalidate나 tag 무효화도 가능했습니다.

하지만 이 방식은 캐싱 단위가 fetch가 아니라 함수 실행 결과이기 때문에 캐시 hit/miss 여부를 로그나 Network에서 확인할 수 없어 캐시 동작을 추적하기 어려웠습니다.

### 2.2 최종 해결책: Supabase SDK에 custom fetch를 적용하는 방식

결과적으로 Supabase Client 생성 시점에 Next.js의 캐시 옵션을 포함한 `Custom Fetch`를 주입하는 방식을 선택했습니다. 이 방식은 SDK의 편리함을 유지하면서도 캐시를 적용할 수 있고 네트워크 레벨에서 캐시 동작을 확인할 수 있습니다.

**Custom Fetch 함수 정의**

```ts
export const createFetch = ({ cache, tags, revalidate }: NextCacheOptions) => 
  (url: RequestInfo | URL, init?: RequestInit) =>
    fetch(url, {
      ...init,
      ...(cache && { cache }),
      ...(tags || revalidate ? { next: { tags: tags ?? [], revalidate: revalidate ?? 0 } } : {}),
    });
```

**Supabase Client 생성 시 적용**

```ts
export const createServerSideClient = async (options: NextCacheOptions = {}) => {
  return createServerClient<Database>(
    URL, KEY,
    {
      global: {
        fetch: createFetch({ cache: 'no-store', ...options }), // 기본값 설정 및 옵션 주입
      },
    }
  );
};
```

### 2.3 주보 목록 페이지 적용 및 최적화 성과

Server Client의 캐시 옵션을 활용해 주보 목록 페이지에 캐싱 전략을 적용했습니다. 주보 목록은 데이터 변경이 적어 `cache: 'force-cache'`와 `revalidate: 86400`을 적용했습니다. 86,400초(24시간) 주기로 DB 조회를 최소화하고 하루 단위로 최신 데이터가 유지되도록 설정했습니다.

```tsx
// page.tsx
const supabase = await createServerSideClient({
    cache: 'force-cache',
    revalidate: 86400
  });
const { data, error } = await supabase.getBulletins({ year, page });
```

여기에 경로 기반 무효화(revalidatePath)를 통해 새 주보 업로드 시 모든 파라미터의 캐시를 즉시 갱신하며 항상 최신 데이터를 조회하도록 구현했습니다.

```tsx
// bulletin.actions.ts
export const createBulletinAction = async (formData: FormData) => {
  // ...
  revalidatePath('/news/bulletin');
}
```

**주요 성과 (Server & Network 지표)**

- API 응답 속도: 112ms → 2ms (DB 조회 생략으로 약 50배 단축)
- 서버 응답 대기 시간(TTFB):  104.98ms → 50.53ms
- API 요청: 매 요청마다 발생 → 하루 1회

|구현 결과|
|:-:|
|![스크린 캡처_20260124_210640](\assets\images\2026-01-20-nextjs-supabase-caching\스크린 캡처_20260124_210640.webp)|

### 2.4. 예상치 못한 문제: Dynamic API의 제약

하지만 주보 상세 페이지에 Full Route Cache를 적용하려는 과정에서 단순한 캐싱 설정만으로는 해결할 수 없는 구조적 문제가 드러났습니다. 

<br/>

## 3. 실행 환경에 따라 Supabase Client를 분리해야 했던 이유

Next.js에서 Dynamic Route에 Full Route Cache를 적용하려면 아래의 조건을 만족해야 합니다.

1. `generateStaticParams`를 사용해 빌드 타임에 생성할 경로가 확정되어야 하고
2. 렌더링 과정에서 `cookies`, `headers` 같은 Dynamic Function을 사용하지 않아야 하며
3. 데이터 캐싱이 적용되어야 합니다

주보 상세 페이지는 `[id]` 기반 Dynamic Route였고 발행 이후 거의 수정되지 않는 콘텐츠였기 때문에 Full Route Cache를 적용하기에 적합한 페이지였습니다. 또한 빌드 타임에 전체 주보 ID 목록을 조회할 수 있었기 때문에 `generateStaticParams`를 통해 생성할 페이지 경로를 미리 확정할 수 있었습니다.

```tsx
export async function getAllBulletinIds() {
  const supabase = await createServerSideClient();
  const { data, error } = await supabase
    .from(BULLETIN_BUCKET)
    .select('id')
    .order('date', { ascending: false });

  if (error) throw error;
  return { data, error };
}

// /[id]/page.tsx
export async function generateStaticParams() {
  const { data: allBulletins, error } = await getAllBulletinIds();

  if (error || !allBulletins) {
    console.error('주보 ID 목록을 불러오는 데 실패했습니다:', error?.message);
    return [];
  }

  return allBulletins.map((bulletin) => ({
    id: bulletin.id.toString(),
  }));
}
```

하지만 빌드 과정에서 다음과 같은 에러가 발생했습니다.

```bash
Error: `cookies` was called outside a request scope
Failed to collect page data for /news/bulletin/[id]
```

원인은 `generateStaticParams`가 실행되는 시점에 있었습니다. 이 함수는 빌드 타임에 동작하지만 내부에서 호출한 `createServerSideClient`는 `cookies()`를 참조하고 있었습니다. 빌드 단계에서는 요청(Request)이나 쿠키 정보가 존재하지 않기 때문에 런타임 에러가 발생했고 이로 인해 빌드가 중단되었습니다.

즉, 요청과 쿠키 컨텍스트에 의존하는 client를 빌드 타임에 사용한 것이 문제였습니다. 이 문제를 계기로 실행 환경에 따라 Supabase client의 역할과 사용 범위를 구분하게 되었습니다.

<br/>

## 4. 실행 환경에 따른 Supabase Client 분리

Supabase Client는 실행 환경에 따라 접근 가능한 정보와 제약이 달랐습니다. 브라우저와 서버 요청 처리 환경에서는 쿠키/세션을 통해 사용자를 식별해야 했고 빌드 타임에서는 요청 컨텍스트 없이 정적 데이터 조회만 가능했습니다. 또한 Next.js 캐시를 적용할 수 있는 방식도 실행 환경에 따라 달랐습니다.

이 차이를 기준으로 Supabase client를 실행 환경별로 나누었고 각 client의 특성을 아래와 같이 구분했습니다.

### 4.1. 클라이언트 유형별 비교

| Client 유형    | 실행 환경              | 쿠키/세션 접근 | Next.js 캐시 적용 | 핵심 용도                                       |
| -------------- | ---------------------- | -------------- | ----------------- | ----------------------------------------------- |
| Browser Client | 브라우저               | ○              | ✕                 | 실시간 구독, 사용자 인터랙션(UI)                |
| Server Client  | 서버 런타임(요청 처리) | ○              | △                 | RLS 기반 조회/쓰기, Server Action/Route Handler |
| Static Client  | 빌드 타임 / ISR        | ✕              | ○                 | `generateStaticParams`, 정적 페이지 생성        |
| Admin Client   | 서버 런타임            | ✕              | ✕                 | RLS 우회(관리자/시스템 작업)                    |

### 4.2. 클라이언트별 상세 구현 및 특징

#### 1) Browser Client - 브라우저에서 세션 유지/실시간 처리

```ts
'use client';
let client: ReturnType<typeof createBrowserClient<Database>> | undefined;

export function getSupabaseBrowserClient() {
  if (client) return client;
  client = createBrowserClient<Database>(...);
  return client;
}
```

- 클라이언트 사이드에서는 페이지 이동과 리렌더링이 잦습니다. 인스턴스를 매번 생성하면 Realtime 연결 중복 및 리소스 낭비가 발생하므로 하나의 인스턴스를 재사용합니다.

- 서버 렌더링 결과에 영향을 주지 않으므로 Next.js의 서버 캐싱을 적용하지 않습니다.

#### 2) Server Client - 서버 런타임(요청 처리)에서 쿠키 기반 RLS 조회

```tsx
export const createServerSideClient = async (options = {}) => {
  const cookieStore = await cookies();

  return createServerClient(..., {
    global: { fetch: createFetch({ cache: 'no-store', ...options }) },
    cookies: { /* getAll / setAll 구현 */ }
  });
};
```

- Server Component, Server Action, Route Handler에서 사용됩니다.
- 쿠키를 통해 사용자를 식별하고 RLS(Row Level Security)가 적용된 데이터를 조회합니다. 사용자별로 결과가 다르므로 기본적으로 `no-store`를 적용하고 필요시 부분적 캐싱을 적용합니다.

#### 3) Static Client - 빌드 타임/ISR에서 정적 페이지 생성

```tsx
export const createStaticClient = (options: NextCacheOptions = {}) => {
  return createClient<Database>(..., {
    global: { fetch: createFetch({ cache: 'force-cache', ...options }) },
    auth: { persistSession: false, ... }
  });
};
```

- 빌드 타임이나 ISR 시점에 실행됩니다.
- `force-cache`를 통해 API 응답을 캐싱하여 빌드 속도와 페이지 로드 성능을 극대화합니다.

#### 4) Admin Client - 보안 정책(RLS) 우회

```tsx
export const createAdminServerClient = (): SupabaseClient<Database> => {
  return createClient<Database>(..., SERVICE_ROLE_KEY, {
    auth: { persistSession: false, ... }
  });
};
```

- `SERVICE_ROLE_KEY`를 사용하여 모든 RLS를 무효화하고 데이터베이스 전체에 접근합니다.
- 키 노출 시 심각한 보안 사고로 이어지므로 반드시 서버 환경에서만 호출하며 데이터 무결성을 위해 캐싱은 사용하지 않습니다.

### 4.3. Client 분리 이후 남은 문제

Client를 분리하면서 빌드 타임 제약과 실행 환경별 요구사항은 해결할 수 있었지만 `custom fetch` 적용 이후 캐시 옵션이 Client 생성 시점에 결정되면서 다른 문제가 남았습니다. 호출부에서 환경에 맞춰 Client를 직접 생성하기 시작하면 동일한 쿼리 로직이 실행 컨텍스트별로 반복되기 쉽고 캐시 설정까지 함께 분기되어 변경 시 수정 지점이 늘어났습니다.

예를 들어 주보 상세 조회는 쿼리 자체는 동일하지만 상세 페이지에서는 Full Route Cache를 위해 Static Client를 사용해야 했고 수정 페이지에서는 세션 기반 동작을 위해 Browser Client를 사용해야 했습니다. 

```tsx
// ISR 상세 페이지용 (Static Client + 캐시 옵션)
export const fetchBulletinDetailStatic = (id: string) => {
  const supabase = createStaticClient({ tags: [`bulletin-${id}`], revalidate: 86400 });
  return supabase.from('bulletins').select('*').eq('id', Number(id)).single();
};

// CSR 수정 페이지용 (Browser Client + 세션)
export const fetchBulletinDetailBrowser = (id: string) => {
  const supabase = getSupabaseBrowserClient();
  return supabase.from('bulletins').select('*').eq('id', Number(id)).single();
};
```

<br/>

## 5. Service Layer 도입: 쿼리 로직과 캐시 설정 분리

Client를 실행 환경별로 분리한 이후에도 호출부에서 직접 캐시 옵션과 Client를 조합해야 하는 문제는 남아 있었습니다. 동일한 데이터에 서로 다른 캐시 정책이 적용되거나 쿼리 변경과 캐시 변경이 함께 발생하는 구조는 관리 부담이 컸습니다.

이를 해결하기 위해 캐시 정책, 쿼리 로직, 실행 환경 조합을 각각 분리하는 Service Layer를 도입했습니다.

### 5.1 캐시 정책 정의

태그와 revalidate 값을 호출부마다 지정하면 관리가 어렵고 동일 데이터에 서로 다른 캐시 정책이 적용될 수 있습니다. 이를 방지하기 위해 캐시 정책을 고정된 규칙으로 정의했습니다.

```ts
// services/bulletin/bulletin-cache.ts
const ROOT = 'bulletin';

export const bulletinCache = {
  list: () => ({ tags: [ROOT, 'bulletin-list'], revalidate: 86400 }),
  detail: (id: string | number) => ({
    tags: [ROOT, 'bulletin-detail', `bulletin-detail-${id}`],
    revalidate: 86400
  }),
  // ...
} as const;
```

### 5.2 쿼리 로직(Service)

Service Layer는 Client를 전달받아 데이터베이스 쿼리만 수행합니다.

```ts
// services/bulletin/bulletin-service.ts
export const bulletinService = (supabase: SupabaseClient<Database>) => ({
  fetchBulletinList: async ({ year, page = 1, limit = 10 }: BulletinParams = {}) => {
    let query = supabase.from('bulletins').select('*', { count: 'exact' });
    if (year) query = query.gte('date', `${year}-01-01`).lte('date', `${year}-12-31`);
    
    return await query.range((page - 1) * limit, page * limit - 1);
  },
  // ... fetchBulletinDetailById, fetchAllBulletinIds 등
});
```

### 5.3 실행 환경 조합(Interface)

최종적으로 서버 컴포넌트나 페이지에서 사용할 인터페이스입니다. Client와 캐시 정책을 조합하여 실제 데이터를 페칭합니다.

```ts
export const fetchBulletinList = (params: BulletinParams = {}) => {
  const supabase = createStaticClient(bulletinCache.list());
  return bulletinService(supabase).fetchBulletinList(params);
};

export const fetchBulletinDetailById = (id: string) => {
  const supabase = createStaticClient(bulletinCache.detail(id));
  return bulletinService(supabase).fetchBulletinDetailById(id);
};
```

쿼리 변경은 Service에서만, 캐시 정책 변경은 Cache 정의에서만 발생하며 페이지는 데이터 조회를 위한 단일 함수만 호출합니다. 이 구조를 통해 데이터 페칭, 캐싱, 실행 환경에 대한 책임을 명확히 분리할 수 있었습니다.


