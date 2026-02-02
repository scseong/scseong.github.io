---
layout: single
title: Next.js 캐싱으로 웹 서버 성능 최적화 (+ Supabase SDK)
category: TIL
toc: true
toc_sticky: true
---

# SSR에서 Full Route Cache까지: 주보 페이지 캐싱 전환 과정

교회 웹사이트에는 매주 업로드되는 주보(교회 소식지)를 제공하는 페이지가 있습니다. 데이터베이스로 Supabase를 사용하고 있으며 초기에는 SSR 방식으로 구현했습니다.

주보는 일주일간 내용이 거의 변하지 않습니다. 하지만 SSR은 매 요청마다 동일한 DB 조회와 HTML 렌더링을 반복해 불필요한 서버 부하가 발생했습니다.

이 글은 Supabase를 사용하는 Next.js 프로젝트에서 주보 상세 페이지에 Full Route Cache를 적용한 경험을 다룹니다. 특히 Supabase SDK와 Next.js 캐싱의 통합, 빌드/런타임 환경 차이로 인한 문제와 해결 과정을 공유합니다.

> **기술 스택**
>Next.js 16+ (App Router) / Supabase / TypeScript

## 1. 문제 인식

### 1.1 SSR의 비효율

주보 상세 페이지를 처음 구현할 때는 세 가지 요구사항을 만족해야 했습니다.

- SEO와 SNS 공유를 위한 초기 요청 시점의 완성된 HTML 제공
- 새 주보 업로드 시 즉시 반영
- 모바일 환경에서도 빠른 초기 로딩 속도

이런 요구사항 때문에 SSR을 선택했습니다. 요청이 들어올 때마다 서버에서 최신 데이터를 조회하고 HTML을 생성해 반환하면 모든 조건을 충족할 수 있었습니다.

하지만 주보는 매주 1회 업로드되며 게시 이후에는 거의 수정되지 않습니다. 그럼에도 불구하고 페이지가 조회될 때마다 서버에서는 동일한 주보 데이터를 다시 조회하고 동일한 HTML을 매번 생성하고 있었습니다.

### 1.2 Full Route Cache 선택

서버 렌더링은 웹 서버 리소스를 많이 사용하는 작업이기 때문에 이 과정을 요청마다 반복하는 방식은 서버 부하로 이어집니다. 이러한 서버 부하는 웹 서버의 요청 처리 속도에 영향을 주며 이는 웹 페이지 로딩 속도와도 직접적으로 연결됩니다.

서버 부하를 줄이기 위한 방법을 찾기 위해 페이지의 특성을 살펴봤습니다.

1. 주보는 매주 1회 업로드되어 페이지 수가 많지 않습니다.
2. 게시 이후에는 내용 변경이 거의 없습니다.
3. 외부 공유를 통해 특정 페이지에 대한 접근이 집중됩니다.

이러한 특성을 종합해보면 트래픽은 특정 주보 상세 페이지에 집중되고 반환되는 결과도 동일합니다. 이에 따라 서버 렌더링 결과를 재사용하는 방식으로 접근하게 되었습니다.

그 결과 Next.js의 **Full Route Cache**를 적용하게 되었습니다. 빌드 시점에 HTML과 RSC Payload를 생성해 서버에 저장해두고 런타임에는 캐시된 결과를 즉시 반환하는 방식입니다.

|                       주보 상세 페이지                       |                       주보 목록 페이지                       |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| ![image-20260121190945365](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260121191012812.png) | ![image-20260121191012812](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260121190945365.png) |

### 1.3 Full Route Cache 적용 조건

페이지가 접속 요청 시점마다 다른 결과를 반환할 수 있는 경우 해당 페이지는 동적으로 렌더링됩니다. Full Route Cache를 적용하려면 페이지가 정적으로 생성될 수 있는 조건을 만족해야 합니다.

1. Dynamic Routes의 경우 `generateStaticParams`로 빌드할 경로를 정의해야 합니다.
2. `cookies()`, `headers()`, `searchParams` 같은 Dynamic API를 사용하면 안 됩니다. 
3. 모든 데이터 페칭이 캐싱되어야 합니다.

따라서 Full Route Cache를 적용하기 위해서는 Dynamic Routes에 ISR을 적용하고 Dynamic API를 제거해야 했습니다. 여기에 데이터 페칭에도 캐싱을 적용할 필요가 있었습니다.

## 2. Supabase SDK에 Next.js 캐싱 적용

### 2.1 Supabase SDK의 캐싱 문제

주보 페이지의 모든 데이터 조회는 Supabase SDK를 통해 이루어집니다. Next.js는 `fetch` 함수에 `cache`와 `next` 옵션을 전달하여 캐싱 전략을 제어할 수 있습니다.

```ts
fetch('https://api.example.com/data', {
  cache: 'force-cache',
  next: { revalidate: 3600, tags: ['data'] }
});
```

Supabase SDK는 내부적으로 `fetch`를 사용하지만 이러한 Next.js 캐시 옵션을 직접 전달할 수 있는 방법이 없었습니다. SDK의 쿼리 메서드에는 캐시 관련 옵션이 존재하지 않았습니다.

### 2.2 Custom Fetch를 통한 해결

캐싱 전략을 적용하기 위한 세 가지 대안을 검토했습니다.

#### 방법 1: Supabase REST API 직접 호출

Supabase는 PostgREST 기반 REST API를 제공합니다. `fetch`를 직접 사용하면 Next.js 캐시 옵션을 자유롭게 설정할 수 있습니다.

```ts
await fetch(
  `${NEXT_PUBLIC_SUPABASE_URL}/rest/v1/bulletins?year=eq.2026&order=date.desc&limit=10`,
  { next: { tags: ['bulletin'], revalidate: 86400 } }
);
```

하지만 SDK가 제공하는 타입 안전성과 쿼리 빌더를 포기해야 했습니다. 모든 쿼리를 직접 작성하고 타입을 수동으로 관리해야 하는 것은 유지보수 비용이 너무 컸습니다.

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

하지만 모든 쿼리 함수를 래핑해야 하고 캐시 hit/miss를 네트워크 레벨에서 확인할 수 없어 디버깅이 어려웠습니다.

#### 방법 3: Custom Fetch 주입 (선택)

Supabase Client 생성 시 `global.fetch`에 Custom Fetch를 주입하는 방식입니다. SDK의 타입 안전성과 쿼리 빌더를 유지하면서도 모든 요청에 Next.js 캐시 옵션을 적용할 수 있어 이 방법을 선택했습니다.

```ts
export const createFetch = ({ cache, tags, revalidate }: NextCacheOptions) => 
  (url: RequestInfo | URL, init?: RequestInit) =>
    fetch(url, {
      ...init,
      ...(cache && { cache }),
      ...(tags || revalidate ? { next: { tags: tags ?? [], revalidate: revalidate ?? 0 } } : {}),
    });
```

Supabase Client 생성 시 Custom Fetch를 주입했습니다.

```ts

export const createServerSideClient = async (options = {}) => {
  const cookieStore = await cookies();
  
  return createServerClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
    global: { fetch: createFetch({ cache: 'no-store', ...options }) },
    cookies: {
      getAll() { return cookieStore.getAll(); },
      setAll(cookiesToSet) { /* ... */ }
    },
  });
};
```

이제 데이터 조회 시 캐시 옵션을 전달할 수 있게 되었습니다.

```ts
const supabase = await createServerSideClient({
  cache: 'force-cache',
  revalidate: 86400,
  tags: ['bulletin', `bulletin-${id}`]
});
```

<br/>

## 3. 빌드 전용 Client 분리

### 3.1 빌드 환경과 런타임 환경의 차이

데이터 캐싱은 적용했지만 Full Route Cache를 위해서는 `generateStaticParams`를 구현해야 했습니다. 주보 상세 페이지는 Dynamic Route(`/bulletin/[id]`)이기 때문에 빌드 시점에 어떤 경로들을 생성할지 정의해야 합니다.

여기서 실행 환경의 차이를 고려할 필요가 있었습니다. `generateStaticParams`는 빌드 시점에 실행되는데 이때는 사용자 요청이 없어 쿠키가 존재하지 않습니다. 반면 런타임에서는 사용자 요청을 처리하므로 쿠키로 사용자를 식별하고 RLS를 적용해야 합니다.

기존의 `createServerClient`는 런타임 환경을 전제로 `cookies()`를 호출합니다. 이 Client를 빌드 시 사용하면 오류가 발생합니다.

```bash
Error: `cookies` was called outside a request scope
Failed to collect page data for /news/bulletin/[id]
```

### 3.2 Static Client 구현

빌드 시점에서 안전하게 동작할 수 있는 별도의 Client가 필요했습니다. 

```ts
import { createClient } from '@supabase/supabase-js';

export const createStaticClient = (options = {}) => {
  return createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
    global: { fetch: createFetch({ cache: 'force-cache', ...options }) },
    auth: { 
      persistSession: false,
      autoRefreshToken: false,
      detectSessionInUrl: false
    }
  });
};
```

Static Client는 `generateStaticParams`와 같은 빌드 시점에 실행되는 함수에서 사용됩니다. 사용자 인증이 필요 없기 때문에 쿠키와 세션 관리를 비활성화했고 기본 캐시 정책은 `force-cache`입니다. 이는 빌드 시점에 조회한 데이터를 캐싱하여 동일한 데이터를 여러 번 조회할 때 네트워크 요청을 줄이기 위함입니다.

Static Client를 사용하는 데이터 조회 함수를 만들었습니다.

```ts
export async function getAllBulletinIds() {
  const supabase = createStaticClient();
  const { data, error } = await supabase .from('bulletins') .select('id').order('date', { ascending: false });
    
  if (error) throw error;
  return { data };
}

export async function getBulletinDetailById(id: string) {
  const supabase = createStaticClient({
    tags: ['bulletin', `bulletin-${id}`]
  });
  
  const { data, error } = await supabase.from('bulletins').select(`*, profiles:profiles ( user_name )`).eq('id', id).single();
    
  if (error) throw error;
  return { data };
}
```

## 4. Full Route Cache 적용해 웹 서버 성능 최적화

### 4.1 Full Route Cache 적용

`generateStaticParams`에서 최근 10개만 빌드하도록 제한했습니다. 모든 주보를 빌드 시점에 생성하면 빌드 시간이 길어지고 오래된 주보는 접근 빈도가 낮기 때문입니다. 나머지 주보는 첫 요청 시 On-Demand ISR로 생성되어 이후 요청부터 캐시됩니다.

```tsx
// app/news/bulletin/[id]/page.tsx
export async function generateStaticParams() {
  const { data: allBulletins } = await getAllBulletinIds();
  
  return allBulletins.slice(0, 10).map((bulletin) => ({
    id: bulletin.id.toString(),
  }));
}

export const revalidate = 86400; // 24 hours

export default async function BulletinDetailPage({ params }: Props) {
  const { id: bulletinId } = await params;
  const [bulletinRes, prevNextRes] = await Promise.all([
    getBulletinDetailById(bulletinId),
    getNavigationBulletins(bulletinId)
  ]);

  const { data: bulletin, error: bulletinError } = bulletinRes;
  const { data: prevNextBulletin, error: navError } = prevNextRes;

  if (!bulletin || bulletinError || navError) notFound();

  return <BulletinDetail bulletin={bulletin} prevNext={prevNextBulletin} />;
}
```

두 데이터 페칭 함수 모두 Static Client를 사용합니다. 이로 인해 빌드 시점에 데이터 조회와 캐싱이 완료되고 런타임에는 데이터 요청이 발생하지 않습니다.

### 4.2 빌드 결과

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

사용자가 `/news/bulletin/98`에 접속하면 Full Route Cache에서 미리 생성된 HTML을 즉시 반환합니다. 빌드 시 생성되지 않은 페이지(예: `/news/bulletin/50`)에 처음 접속하면 서버에서 데이터를 페칭하고 렌더링하여 HTML을 생성합니다. 생성된 HTML을 Full Route Cache에 저장하고 이후 동일한 페이지 요청은 캐시된 HTML을 반환합니다.

### 4.3 성능 측정 및 개선 결과

Vercel Toolbar의 Page Tracing 기능을 활용하여 Preview 환경에서 TTFB를 측정했습니다. Logs에서 확인한 결과는 다음과 같습니다.

| 측정                           | 개선 전 (SSR) | 개선 후 (SSG) | 캐시 상태        |
| ------------------------------ | ------------- | ------------- | ---------------- |
| 1회 (초기 접속)                | 914ms         | 8.73s         | MISS / PRERENDER |
| 2회                            | 191ms         | 103.12ms      | MISS / HIT       |
| 3회                            | 171ms         | 33.17ms       | MISS / HIT       |
| 4회                            | 203ms         | 38.15ms       | MISS / HIT       |
| 5회                            | 297ms         | 75.54ms       | MISS / HIT       |
| 6회                            | 134ms         | 32.74ms       | MISS / HIT       |
| **평균**<br />(초기 접속 제외) | **199.2ms**   | **56.48ms**   | -                |

첫 번째 요청은 빌드 시 생성되지 않은 페이지에 대한 On-Demand ISR로 8.73초가 소요되었습니다. 하지만 두 번째 요청부터는 Full Route Cache가 적용되어 평균 56.48ms로 응답했습니다.

개선 전 SSR은 모든 요청이 `X-Vercel-Cache: MISS`로 매번 서버 렌더링이 발생했습니다. 개선 후에는 첫 요청 이후 `X-Vercel-Cache: HIT`로 캐시된 HTML을 즉시 반환했습니다.

| 개선 전 (SSR)                                                | 개선 후 (ISR)                                                |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image-20260202212705112](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260202212705112.png) | ![image-20260202212713586](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260202212713586.png) |



**개선 성과**

| 항목              | 개선 전 (SSR)          | 개선 후 (ISR)                    |
| ----------------- | ---------------------- | -------------------------------- |
| TTFB (평균)       | 199.2ms                | **56.48ms (71.6%↓)**             |
| 생성 및 갱신 주기 | 매 요청 시 실시간 생성 | 빌드 시 생성 + 24시간 주기 갱신  |
| 갱신 전략         | 실시간 생성            | 수동 즉시 갱신 + 24시간마다 갱신 |
| DB 조회           | 모든 요청마다 실행     | 갱신 주기 내 1회 실행            |

## 5. 결론

주보 상세 페이지는 데이터 변경 빈도가 낮음에도 SSR 방식으로 구현되어 동일한 페이지 요청마다 DB 조회와 HTML 렌더링이 반복되고 있었습니다.

이에 페이지 특성에 맞춰 서버에서 생성한 결과를 재사용할 수 있도록 Full Route Cache를 적용했습니다. 이 과정에서 Supabase SDK에는 Custom Fetch를 주입했고 빌드 시점과 런타임의 요구사항에 따라 Client를 분리해 사용했습니다.

그 결과 평균 TTFB가 199ms에서 56ms로 약 71% 감소했으며 요청마다 수행되던 DB 조회와 HTML 렌더링이 요청 경로에서 생략되어 서버 부하 역시 크게 줄었습니다.

이 과정에서 빌드 타임과 런타임의 차이, Supabase Client의 환경별 동작, Next.js 캐싱 메커니즘의 연결 관계를 명확히 이해하게 되었습니다. 단순히 동작하는 코드를 넘어서 실행 환경을 고려한 아키텍처 설계의 중요성을 체감한 경험이었습니다.
