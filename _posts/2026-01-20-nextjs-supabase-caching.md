---
layout: single
title: 주보 페이지로 설계한 Next.js 데이터 페칭 최적화 및 캐싱 전략
category: TIL
toc: true
toc_sticky: true
---

# 주보 페이지로 설계한 Next.js 데이터 페칭 최적화 및 아키텍처 개선

교회 웹사이트에서 **주보 페이지**는 성도들이 가장 자주 찾는 핵심 콘텐츠이자, 외부인들에게 교회의 소식을 전하는 중요한 창구입니다. 주보는 보통 주 1회 발행되며 이후에는 수정이 거의 없지만 새로운 주보가 업로드되면 즉시 반영되어야 하고 외부 공유가 잦다는 특징이 있습니다. 초기에는 항상 최신 데이터를 제공하기 위해 모든 요청에 대응하는 **SSR(Server-Side Rendering)** 방식을 채택했으나 데이터 변화가 적음에도 매 요청마다 동일한 데이터베이스 조회가 반복되어 서버 리소스가 낭비되는 비효율을 확인했습니다.

이를 해결하기 위해 Supabase와 Next.js의 캐싱 메커니즘을 결합하고 실행 환경에 따라 클라이언트를 분리하며 Service Layer를 도입한 과정을 정리했습니다.

> 사용한 기술들은 다음과 같습니다.
>
> - **Framework:** `Next.js 16 (App Router)`
> - **Database:** `Supabase`
> - **Language:** `TypeScript`



<br/>

## 1. 주보 페이지 최적화: 아키텍처 설계

주보 페이지는 교회 소식을 전하는 창구로서 기술적으로 다음과 같은 요구사항을 충족해야 합니다.

- 검색 노출과 SNS 공유를 위해 서버 측에서 완성된 HTML과 메타데이터 제공
- 주 1회 업데이트 후 거의 수정되지 않지만 새 주보가 업로드되면 즉시 반영되어야 함
- 특정 시간대 모바일 접속이 급증하는 환경에서도 안정적이고 빠른 응답 속도를 보장

위 세 가지 목표를 동시에 충족하기 위해 처음 선택한 것은 실시간 생성 방식(SSR)이었습니다. 사용자가 접속하는 순간 서버가 페이지를 생성하므로 완성된 HTML을 제공하면서도 데이터의 최신성을 보장할 수 있었기 때문입니다.

하지만 주보 데이터는 일주일간 변하지 않는 정적인 특성을 가집니다. 데이터 변화가 없음에도 매 접속마다 DB 조회와 렌더링을 반복하는 SSR 방식은 불필요한 서버 자원을 소모하는 비효율이 있었습니다.

이러한 문제를 해결하고자 미리 생성된 결과물을 재사용하는 정적 렌더링(ISR) 방식으로의 전환을 고려했습니다. 

|                       주보 목록 페이지                       |                       주보 상세 페이지                       |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| ![image-20260121191012812](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260121190945365.png) | ![image-20260121190945365](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260121191012812.png) |


### ① 주보 목록 페이지: 동적 렌더링 유지와 데이터 캐싱

주보 목록은 연도 필터링과 페이지네이션 관리를 위해 Query Parameter를 사용합니다. 이때 서버 컴포넌트에서 `searchParams`에 접근하게 되는데 이 Dynamic API를 사용하는 순간 해당 페이지는 자동으로 동적 렌더링(Dynamic Rendering) 대상이 됩니다.

```tsx
export default async function BulletinPage({ searchParams }: Props) {
  const { year, page } = await searchParams;
  // ... 
}
```

주보 목록 페이지를 정적 페이지로 전환하려면 기존의 `/bulletin?year=2026` 같은 쿼리 파라미터 구조를 `/bulletin/year/2026` 형태의 Static Route로 변경해야만 했습니다. 하지만 모든 필터 조합을 정적 페이지로 미리 생성하는 것은 빌드 타임과 관리 비용 면에서 비효율적이라 판단했습니다.

따라서 목록 페이지는 동적 렌더링을 유지하되 데이터 조회 결과만 캐싱(Data Cache)하는 방식을 적용했습니다 .

### ② 주보 상세 페이지: Full Route Cache를 통한 성능 극대화

반면 상세 페이지는 발행 후 수정이 거의 없고 외부 공유가 빈번한 특성상 SEO 최적화와 즉각적인 응답 속도가 무엇보다 중요했습니다.

이러한 이유로 접속 시점에 페이지를 만드는 대신 빌드 시점에 미리 HTML을 생성하는 정적 생성(Static Generation) 방식을 채택했습니다. 이렇게 생성된 페이지는 서버의 Full Route Cache에 보관되어 요청 시 추가 연산 없이 캐시된 결과물을 즉시 반환하므로 사용자에게 빠른 응답 속도를 제공합니다.

![Diagram showing the default caching behavior in Next.js for the four mechanisms, with HIT, MISS and SET at build time and when a route is first visited.](https://nextjs.org/_next/image?url=https%3A%2F%2Fh8DxKfmAPhn8O0p3.public.blob.vercel-storage.com%2Fdocs%2Fdark%2Fcaching-overview.png&w=3840&q=75)

<br/>

## 2. Supabase SDK 환경에서 Next.js 캐시를 적용하는 방법

주보 목록 페이지의 데이터 캐싱과 상세 페이지의 Full Route Cache를 적용하려면, 데이터 요청이 Next.js가 확장한 `fetch` 를 통해 실행되어야 했습니다. `cache`, `revalidate`, `tags` 같은 캐시 옵션은 이 `fetch` 호출에만 적용되기 때문입니다.

하지만 기존 구조에서는 모든 데이터 조회가 Supabase SDK를 통해 이루어지고 있었습니다. Supabase SDK는 내부적으로 HTTP 요청을 수행하지만 이 요청에 Next.js의 캐시 옵션을 전달할 수 있는 인터페이스를 제공하지 않습니다.

이 문제를 해결하기 위해 데이터 조회에 Next.js 캐시를 적용할 수 있는 방법들을 시도했습니다.

### ① Supabase REST API를 fetch로 직접 호출

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

### ② unstable_cache를 사용하는 방식

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

이 방식의 장점은 Supabase SDK를 그대로 사용할 수 있다는 점입니다. 많은 변경 없이 캐싱을 도입할 수 있고 revalidate나 tag 무효화도 가능했습니다.

하지만 이 방식은 캐싱 단위가 fetch가 아니라 함수 실행 결과이기 때문에 캐시 hit/miss 여부를 로그나 Network에서 확인할 수 없어 캐시 동작을 추적하기 어려웠습니다.

### ③ Supabase SDK에 custom fetch를 적용하는 방식

Supabase SDK를 그대로 사용하면서 Next.js의 캐싱 옵션을 적용하기 위해 client 생성 시 `global.fetch`에 custom fetch를 주입하는 방식을 사용했습니다. 이 custom fetch는 cache, revalidate, tags 옵션을 포함한 Next.js의 fetch를 감싸 Supabase SDK 내부 요청에 전달합니다.

```ts
export const createFetch =
  ({ cache, tags, revalidate }) =>
  (url: RequestInfo | URL, init?: RequestInit) =>
    fetch(url, {
      ...init,
      ...(cache ? { cache } : {}),
      ...(tags || revalidate
        ? { next: { tags: tags ?? [], revalidate: revalidate ?? 0 } }
        : {}),
    });
```

이 fetch는 Supabase client 생성 시점에 주입됩니다.
```ts
export const createServerSideClient = async (
  options: NextCacheOptions = {}
) => {
  const cookieStore = await cookies();
  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      global: {
        fetch: createFetch({ cache: 'no-store', ...options }),
      },
      // cookies, auth 등 기타 설정
    }
  );
};
```

```tsx
const supabase = await createServerSideClient({
    cache: 'force-cache',
    tags: ['bulletins'],
    revalidate: 86400
  });
const { data, error } = await supabase.getBulletins({ year, page });
```

결과적으로 Supabase SDK에 custom fetch를 주입하는 방식을 선택했습니다. 이 방식은 기존 Supabase SDK 기반 데이터 조회 로직을 그대로 유지하면서도 데이터 요청 단계에서 revalidate와 tags 기반 캐싱을 명시적으로 제어할 수 있었기 때문입니다. 또한 캐시 hit 여부를 서버 로그와 Network 레벨에서 확인할 수 있어 캐싱 동작을 검증하기도 쉬웠습니다.

캐싱 전략을 주보 목록 페이지에 먼저 적용했습니다. 동일한 요청이 반복될 때 Supabase 호출이 캐시에서 처리되었고 평균 응답 시간이 눈에 띄게 감소했습니다.

| 구분          | 캐시 미적용                                                  | 캐시 적용                                                    |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| TTFB          | ![image-20260122211705461](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260122211705461.png) | ![image-20260122211651643](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260122211651643.png) |
| API 응답 속도 | GET .../rpc/getbulletinSumarry 200 in **112ms** (cache skip) | GET .../rpc/getbulletinSumarry 200 in **2ms** (cache hit)    |
| 상태          | DB를 직접 조회함                                             | 저장된 결과값을 즉시 반환                                    |

하지만 주보 상세 페이지의 Full Route Cache에 적용하려는 과정에서 단순한 캐싱 설정만으로는 해결할 수 없는 구조적 문제가 드러나기 시작했습니다. 이 문제는 이후 Supabase client를 실행 환경 기준으로 분리해야 했던 이유로 이어집니다.

<br/>

### 3. 실행 환경에 따라 Supabase client를 분리해야 했던 이유

Next.js에서 Dynamic Route에 Full Route Cache를 적용하려면 몇 가지 전제 조건을 만족해야 합니다.

1. generateStaticParams를 사용해 빌드 타임에 생성할 경로가 확정되어야 하고
2. 렌더링 과정에서 cookies, headers 같은 Dynamic Function을 사용하지 않아야 합니다

주보 상세 페이지는 `[id]` 기반 Dynamic Route였고 주보는 발행 이후 거의 수정되지 않는 콘텐츠였기 때문에 Full Route Cache를 적용하기에 적합한 구조였습니다. 또한 빌드 타임에 전체 주보 ID 목록을 조회할 수 있었기 때문에 `generateStaticParams`를 통해 생성할 페이지 경로를 미리 확정할 수 있었습니다.

```tsx
export async function generateStaticParams() {
  const supabase = await createServerSideClient();
  const { data: allBulletins, error } = await supabase.getAllBulltinIds();

  if (error || !allBulletins) {
    console.error('주보 ID 목록을 불러오는 데 실패했습니다:', error?.message);
    return [];
  }

  return allBulletins.map((bulletin) => ({
    id: bulletin.id.toString(),
  }));
}
```

Full Route Cache의 조건을 모두 충족하는 것처럼 보였지만 빌드 과정에서 다음과 같은 에러가 발생했습니다.

```bash
Error: `cookies` was called outside a request scope
Failed to collect page data for /news/bulletin/[id]
```

이 에러의 원인은 `generateStaticParams` 내부에서 사용한 `createServerSideClient`에 있었습니다. 이 client는 HTTP 요청이 존재하는 환경에서만 동작하도록 설계되어 내부에서 `cookies()`를 호출합니다.

하지만 `generateStaticParams`는 HTTP 요청과 무관한 빌드 타임에 실행됩니다. 이 시점에는 쿠키나 요청 정보가 존재하지 않기 때문에 `cookies()`가 호출되는 순간 Next.js는 이를 잘못된 실행 환경으로 판단하고 빌드를 중단합니다.

결과적으로, 빌드 타임에서 실행 가능한 client와 요청 시점에 사용해야 하는 client를 구분해야 할 필요성이 드러났습니다.

<br/>

### 4. 실행 환경을 기준으로 Supabase client를 분리





<br/>

<br/>

<br/>

<br/>

<br/>

<br/>

<br/>

<br/>

<br/>

<br/>

<br/>

<br/>

<br/>

교회 웹사이트 주보 페이지는 수정 빈도가 낮지만 새로운 주보 발행 시 즉시 반영이 필요한 콘텐츠입니다. SEO를 위해 목록과 상세 페이지를 SSR로 구현했으나 매 요청마다 반복되는 데이터 조회로 서버 리소스가 불필요하게 소모되었습니다.


이 문제를 해결하기 위해 캐싱을 시도했으나 기존 Supabase client 구조로는 Next.js의 fetch 옵션을 직접 제어할 수 없었습니다. 이에 실행 환경별 클라이언트 분리와 Service Layer를 도입하여 캐싱 메커니즘을 활용할 수 있는 구조를 설계했습니다. 렌더링 방식에 따른 캐싱 적용과 데이터 무효화 전략을 고민하며 서버 부하를 줄여나갔던 과정을 정리해 보았습니다.

본격적인 내용에 앞서 이번 프로젝트에서 사용한 기술들은 다음과 같습니다.

- **Framework:** `Next.js 16 (App Router)`
- **Database:** `Supabase`
- **Language:** `TypeScript`



### 주보 페이지 기능과 초기 문제 상황

주보 페이지는 교회 웹사이트에서 가장 조회 빈도가 높은 콘텐츠 중 하나입니다. 특히 공유가 잦은 페이지로 SEO와 초기 응답 속도가 중요했고 다음과 같은 특징을 가지고 있었습니다.

- 주보는 주 1회 발행, 발행 이후에는 거의 수정되지 않음
- 새 주보가 업로드되면 즉시 반영되어야 함
- 목록 페이지와 상세 페이지 모두 외부 공유 가능성이 높음

| 주보 목록 페이지 |       주보 상세 페이지                      |           
| • 최신 주보 섹션: 항상 최신 주보 노출 <br />• 연도 기준 필터: 원하는 시점 탐색 가능 <br />• 페이지네이션: 목록형 탐색 제공 | • 동적 라우팅: `[id]` 기반의 개별 페이지 <br />• 이동 네비게이션: 이전/다음 주보 바로가기 <br />• SEO 최적화: 높은 공유 빈도와 검색 노출 대응 |
| ![image-20260121191012812](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260121190945365.png) |![image-20260121190945365](\assets\images\2026-01-20-nextjs-supabase-caching\image-20260121191012812.png) |

초기에는 서버에서 HTML을 생성해 SEO에 유리하고 공유 시점에 항상 최신 데이터를 제공할 수 있다는 점에서 SSR로 구현했습니다. 다만 주보 데이터는 일주일에 한 번만 변경되고 발행 이후에는 거의 수정되지 않는 콘텐츠였습니다. 그럼에도 요청마다 동일한 Supabase 쿼리가 반복 실행되면서 서버 리소스가 불필요하게 사용되어 최적화하기로 했습니다.

### 렌더링 방식을 다시 고민하다 – 주보 목록 및 상세 페이지

주보 목록 페이지는 연도와 페이지네이션을 Query Parameter로 관리하고 있었고 `searchParams`를 사용해 동적으로 렌더링되는 페이지였습니다.

```tsx
export default async function BulletinPage({ searchParams }: Props) {
  const { year, page } = await searchParams;
  ...
}
```

정적 페이지(ISR)로 전환하는 방안도 검토했지만 이를 위해서는 `/bulletin?year={year}&page={page}`를 `/year/{year}/page/{page}` 같은 static route로 변경해야 했습니다. 이 경우 연도와 페이지네이션 조합에 따라 생성해야 할 정적 페이지 수가 늘어나 빌드 타임과 관리 비용 측면에서 부담이 커질 수밖에 없었습니다.

따라서, 주보 목록 페이지는 렌더링 방식을 정적으로 변경하기보다는 동적 렌더링을 유지하면서 데이터 조회에 캐시를 적용하는 방식이 적합하다고 판단했습니다.

반면 주보 상세 페이지는 한 번 작성되면 거의 수정되지 않고 비로그인 상태에서도 누구나 접근할 수 있는 페이지였습니다. 또한 SEO가 중요하고 빠르게 서빙되는 것이 중요한 페이지였기 때문에 빌드 타임에 미리 생성된 페이지를 활용하는 방식이 더 적절하다고 생각했습니다. 

이런 특성상 상세 페이지는 접속 요청 시 캐시된 페이지를 바로 반환할 수 있는 Full Route Cache를 적용하기로 했습니다.

### Next.js 캐싱을 적용하려다 마주한 Supabase client의 한계

주보 목록 페이지에는 데이터 캐싱이 필요했고 상세 페이지 역시 Full Route Cache를 적용하려면 데이터 페칭 단계부터 캐싱되어야 했습니다. 특히 캐시는 fetch 호출 단위에서 제어되기 때문에 데이터 요청이 fetch를 통해 이루어져야 했습니다.

하지만 기존 구조에서는 데이터 조회가 Supabase client를 통해 이루어지고 있어 cache, revalidate, tags 같은 Next.js의 fetch 기반 캐싱 옵션을 적용할 수 없었습니다. Supabase client 내부에서 요청이 실행되기 때문에 캐싱을 제어할 수 없었기 때문입니다.

Supabase client를 그대로 사용한 상태에서는 Next.js의 캐싱 전략을 적용할 수 없었기 때문에 캐싱을 적용하기 위한 몇 가지 방식을 검토했습니다.

***

1) Supabase REST API를 직접 `fetch`로 호출하는 방식

Supabase client를 사용하지 않고 PostgREST API를 `fetch`로 직접 호출하는 방식입니다.

| **장점** <br />• Next.js `fetch` 캐싱 옵션 제어 가능<br />• `revalidate`, `tags` 기반 무효화 구현이 직관적<br />• 캐싱 흐름이 명확함 | **단점**<br />• Supabase 쿼리 빌더 사용 불가<br />• 타입 안전성 저하<br />• 기존 코드 수정으로 유지보수 비용 증가 |

→ 캐싱 적용은 쉬웠으나 데이터 접근 방식을 크게 변경해야 했습니다.

#### 2) `unstable_cache`를 사용하는 방식

`unstable_cache`를 사용하면 `fetch`가 아니라 서버 함수의 실행 결과 자체를 캐싱할 수 있습니다.


| **장점**<br />• Supabase client 유지 가능<br />• `revalidate`, `tags` 등 캐시 제어 가능<br />• 구조 변경이 적음 |**단점**<br /> • `fetch` 단위 캐싱 아님<br />• 직관적이지 않은 캐시 구조<br />• API의 불안정성 (unstable) |

→ 동작은 가능했지만 캐싱 흐름을 예측하기 어려웠습니다.

**3) Supabase client에 custom fetch 적용**

Supabase 클라이언트를 생성할 때 `fetch` 옵션에 Next.js의 확장된 `fetch`를 주입하는 방식입니다.

| **장점**<br />• 기존 Supabase 쿼리 로직과 타입 안정성 유지<br />• `revalidate`, `tags` 기반 캐싱 가능 | **단점**<br /> •  client 생성 구조가 복잡해짐<br />•  캐싱 옵션을 명시적으로 관리 필요   |



REST API 방식과 unstable_cache는 각각 유지보수 비용과 캐싱 흐름의 명확성 측면에서 아쉬움이 있었습니다. 반면 Supabase client에 custom fetch를 적용하는 방식은 기존 구조를 유지하면서도 Next.js의 fetch 캐싱 전략을 그대로 사용할 수 있었습니다.

### Supabase client에 custom fetch를 적용

Supabase JS client는 생성 시점에 `global.fetch`를 주입할 수 있습니다. 이 지점을 활용해, Next.js의 `fetch` 옵션(`cache`, `revalidate`, `tags`)을 외부에서 주입할 수 있는 구조를 만들었습니다. 이렇게 하면 기존 Supabase 쿼리 로직은 그대로 유지하면서도, 데이터 요청 단계에서 캐싱을 제어할 수 있습니다.

```ts
import { NextCacheOptions } from '@/shared/supabase/types';

export const createFetch = ({ cache, tags, revalidate }: NextCacheOptions = {}) => {
  return (url: RequestInfo | URL, init?: RequestInit) => {
    const next =
      tags || revalidate
        ? { tags: tags ?? [], revalidate: revalidate ?? 0 }
        : undefined;

    return fetch(url, {
      ...init,
      ...(cache ? { cache } : {}),
      ...(next ? { next } : {})
    });
  };
};

```

다음으로 이 `fetch`를 Supabase client 생성 시점에 주입합니다.

```tsx
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';
import { createFetch } from '@/shared/supabase/fetch';

export const createSupabaseClient = async ({
  cache,
  tags,
  revalidate
}: NextCacheOptions = {}) => {
  const cookieStore = await cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get: (key) => cookieStore.get(key)?.value,
        set: (key, value, options) => cookieStore.set(key, value, options),
        remove: (key, options) => cookieStore.set(key, '', options)
      },
      global: {
        fetch: createFetch({ cache, tags, revalidate })
      }
    }
  );
};

```

그 결과, Supabase를 통한 데이터 조회에도 `revalidate`와 `tags` 기반 캐싱을 적용할 수 있게 되었습니다.

페이지에서는 기존과 동일하게 Supabase client를 사용하되, **캐싱 전략만 명시적으로 전달**하면 됩니다.

```tsx
const supabase = await createServerSideClient({
  cache: 'force-cache',
  tags: ['bulletin'],
  revalidate: 60 * 60 * 24
});

const { data, error } = await fetchBulletinSummary({ year, page });
```



데이터 캐싱 적용 전에는 페이지 요청마다 Supabase RPC가 실행되었고, 평균 응답 시간은 약 536ms 수준이었다. custom fetch를 통해 Next.js의 데이터 캐싱을 적용한 이후에는 최초 요청 이후 Supabase RPC가 캐시에서 처리되었고, 평균 응답 시간은 약 315ms로 감소했다.

서버 로그 기준으로도 캐싱 효과가 명확했다. 캐싱 미적용 시 Supabase RPC 호출에 약 123ms가 소요된 반면, 캐시 hit 이후에는 동일 요청이 4ms 내로 처리되었다. 이를 통해 Supabase client를 사용하는 구조에서도 Next.js의 데이터 캐싱 전략이 정상적으로 동작함을 확인할 수 있었다.

```bash
 GET /news/bulletin 200 in 486ms (compile: 20ms, proxy.ts: 12ms, render: 454ms)
 │ GET https://ndimreqwgdiwtjonjjzq.supabase.co/rest/v1/rpc/getbulletinsummary?page=1&limit_count=10 200 in 123ms (cache skip)
 │ │ Cache skipped reason: (cache-control: no-cache (hard refresh))
 GET /news/bulletin 200 in 328ms (compile: 19ms, proxy.ts: 15ms, render: 294ms)
 │ GET https://ndimreqwgdiwtjonjjzq.supabase.co/rest/v1/rpc/getbulletinsummary?page=1&limit_count=10 200 in 4ms (cache hit)
```



> *이 접근법은 Supabase client 내부의 fetch를 Next.js의 fetch로 덮어쓰는 해법에서 영감을 받았습니다 (참고: \*Enhancing Data Caching in NextJS 14 with Supabase Webhooks\*).* 





### 실행 환경에 따라 Supabase client를 분리해야 했던 이유

custom fetch를 적용하면서 Supabase client를 통해서도 Next.js의 캐싱 전략을 사용할 수 있게 되었습니다. 하지만 이 시점에서 또 하나의 문제가 드러났습니다. **모든 Supabase 요청이 동일한 실행 환경에서 동작하지 않는다는 점**이었습니다.

주보 목록 페이지는 SSR 환경에서 실행되며 데이터 캐싱이 필요했고, 상세 페이지는 빌드 타임에 실행되어 Full Route Cache의 대상이 됩니다. 반면 관리자 페이지나 인증이 필요한 요청은 쿠키를 읽고 쓰는 서버 환경에서 동작해야 했고, 브라우저에서 실행되는 client 요청도 별도로 존재했습니다.

문제는 이 서로 다른 요구사항을 **하나의 Supabase client로 처리하려고 했다는 점**이었습니다. 캐싱 옵션, 쿠키 접근 여부, 실행 시점(빌드 타임 / 요청 시점 / 브라우저)에 따라 Supabase client의 설정과 책임이 달라졌고, 하나의 client에 모든 역할을 맡기기에는 무리가 있었습니다.

주보 상세 페이지에는 **Full Route Cache**를 적용하기로 결정했었다.
 주보는 주 1회 발행되고, 발행 이후 수정되는 경우가 거의 없기 때문이다.

### 5.1 Full Route Cache를 적용하기 위한 전제 조건

Next.js에서 Dynamic Route에 Full Route Cache를 적용하려면
 다음 조건을 만족해야 한다.

1. `generateStaticParams`를 사용해 **빌드 타임에 경로를 확정**할 것
2. Dynamic Route에서 **ISR 방식으로 동작**할 것
3. 렌더링 과정에서
    **Dynamic Function (`cookies`, `headers` 등)을 사용하지 않을 것**

주보 상세 페이지는 이 조건에 잘 맞는 구조였다.

- `[id]` 기반 Dynamic Route
- 주보 ID 목록은 빌드 타임에 조회 가능
- 페이지 자체는 모든 사용자에게 동일

그래서 다음과 같이 `generateStaticParams`를 구현했다.

------

### 5.2 `generateStaticParams` 구현

```
export async function generateStaticParams() {
  const supabase = createServerSideClient();
  const { data: allBulletins, error } =
    await supabase.fetchAllBulletinIds();

  if (error || !allBulletins) {
    console.error(
      '주보 ID 목록을 불러오는 데 실패했습니다:',
      error?.message
    );
    return [];
  }

  return allBulletins.map((bulletin) => ({
    id: bulletin.id.toString(),
  }));
}
```

이 코드는 겉보기에는 문제가 없어 보였다.

- 빌드 타임에 전체 주보 ID를 가져오고
- 각 ID에 대해 정적 페이지를 생성한다

하지만 빌드 과정에서 다음과 같은 에러가 발생했다.

------

### 5.3 빌드 에러

```
Error: `cookies` was called outside a request scope.
Read more: https://nextjs.org/docs/messages/next-dynamic-api-wrong-context
```

요약하면:

- 빌드 중 `cookies()`가 호출되었고
- Next.js는 이를 **잘못된 실행 컨텍스트**로 판단했다

결과적으로:

- `.html` 파일 생성 실패
- Full Route Cache 적용 실패
- 상세 페이지가 Dynamic으로 강등됨

------

### 5.4 왜 이런 에러가 발생했을까?

문제의 핵심은
 `generateStaticParams` **자체가 아니라**,
 그 안에서 사용한 **Supabase client**였다.

```
const supabase = createServerSideClient();
```

`createServerSideClient`는 다음을 전제로 설계된 client다.

- 요청(Request)이 존재하고
- 사용자 세션을 다루며
- 내부적으로 `cookies()`를 읽고 / 쓸 수 있다

하지만 `generateStaticParams`는:

- 요청 시점이 아니라
- **빌드 타임에 실행**되며
- Request / Cookie / Header 컨텍스트가 존재하지 않는다

즉, 이 상황은 이렇게 요약할 수 있다.

> **빌드 타임 코드에서
>  요청 기반 Supabase client를 사용하고 있었다**

Next.js 입장에서는:

- `cookies()` 호출 = 요청별 상태 의존
- 요청별 상태 의존 = 정적 생성 불가

로 판단할 수밖에 없었다.

------

### 5.5 Full Route Cache가 깨진 진짜 이유

이 문제는 단순히 “에러가 났다”가 아니라,
 **Full Route Cache의 조건을 위반했기 때문에 발생한 문제**였다.

정리하면:

| 항목                  | 상태             |
| --------------------- | ---------------- |
| Dynamic Route         | ⭕                |
| generateStaticParams  | ⭕                |
| ISR                   | ⭕                |
| Dynamic Function 사용 | ❌ (cookies 사용) |

즉,

> **Full Route Cache를 적용하려는 페이지에서
>  실행 환경에 맞지 않는 client를 사용했다**

는 것이 문제의 본질이었다.





### — Supabase client는 실행 환경에 따라 달라져야 했다

섹션 5의 에러를 통해 한 가지 사실이 명확해졌다.

> **Full Route Cache가 깨진 이유는 캐싱 설정이 아니라,
>  실행 환경에 맞지 않는 Supabase client를 사용했기 때문**이었다.

이 지점에서 문제를 다시 정의할 필요가 있었다.

------

### 6.1 지금까지의 가정이 틀렸다는 깨달음

처음에는 이렇게 생각했다.

- Supabase client는 하나면 충분하다
- 어디서 호출하든 같은 client를 써도 된다
- 캐싱 여부는 옵션으로 해결할 수 있다

하지만 실제로는 그렇지 않았다.

Supabase client는 내부적으로 다음과 같은 **부작용(side effect)**을 가진다.

- 쿠키를 읽는다
- 세션을 갱신하려 한다
- 요청별 상태에 의존한다

이 동작은 **SSR 환경에서는 필요하고 자연스럽다.**
 하지만 **빌드 타임 / SSG 환경에서는 치명적**이다.

------

### 6.2 실행 환경별 요구사항은 완전히 달랐다

문제를 정리해보니,
 Supabase client에 요구되는 조건은 실행 환경마다 달랐다.

#### 빌드 타임 / SSG / Full Route Cache

- 모든 사용자에게 동일한 결과
- 요청 컨텍스트 없음
- 쿠키 접근 ❌
- 세션 관리 ❌
- 정적 캐싱 ⭕ 필수

#### SSR / 요청 기반 페이지

- 사용자별 결과 가능
- 요청 컨텍스트 존재
- 쿠키 접근 ⭕
- 세션 관리 ⭕
- 정적 캐싱 ❌

#### 브라우저 환경

- 사용자 UX 중심
- 클라이언트 생명주기 내 상태 유지
- CSR 전용 client 필요

이 요구사항을 하나의 Supabase client로 만족시키는 것은
 구조적으로 불가능했다.



그래서 결론은 자연스럽게 이 지점에 도달했다.

> **Supabase client는 하나일 수 없다.**

그리고 더 중요한 결론은 이것이었다.

> **client를 나누는 기준은
>  “기능”이 아니라 “실행 환경”이어야 한다.**

- 어떤 기능을 쓰느냐가 아니라
- **어디에서 실행되느냐**가 client의 형태를 결정해야 했다



## Supabase client 분리 전략

### — 기능이 아니라 “실행 환경”을 기준으로 나누다

섹션 6에서 결론은 명확해졌다.

> **Supabase client는 하나일 수 없고,
>  분리 기준은 기능이 아니라 실행 환경이어야 한다.**

이 기준을 코드로 옮기기 위해,
 각 실행 환경에서 **허용되는 부작용**을 먼저 정의했다.

------

### 7.1 분리 기준: “이 환경에서 무엇이 허용되는가?”

Supabase client를 나누기 전에,
 각 환경에서 허용되는 동작을 정리했다.

| 실행 환경       | 쿠키 접근 | 세션 갱신 | 요청별 상태 | 정적 캐시 |
| --------------- | --------- | --------- | ----------- | --------- |
| 빌드 타임 / SSG | ❌         | ❌         | ❌           | ⭕         |
| SSR             | ⭕         | ⭕         | ⭕           | ❌         |
| Browser         | ⭕         | ⭕         | ⭕           | ❌         |
| Admin           | ❌         | ❌         | ❌           | ❌         |

이 표가 이후 client 분리의 **절대 기준**이 되었다.

------

### 7.2 Static Client

#### — 빌드 타임과 Full Route Cache를 위한 client

**의도**

- 모든 사용자에게 동일한 데이터
- 빌드 타임 / SSG / ISR / Full Route Cache 전용
- 정적 캐싱이 절대 깨지면 안 됨

**핵심 원칙**

- 쿠키 접근 ❌
- 세션 관리 ❌
- 요청 컨텍스트 의존 ❌

```
export const createStaticClient = (options) =>
  createClient(SUPABASE_URL, ANON_KEY, {
    global: {
      fetch: createFetch({
        cache: 'force-cache',
        ...options,
      }),
    },
    auth: {
      persistSession: false,
      autoRefreshToken: false,
      detectSessionInUrl: false,
    },
  });
```

이 client는 한 가지 목적만 가진다.

> **정적 페이지가
>  어떤 이유로도 Dynamic으로 강등되지 않게 하는 것**

------

### 7.3 Server Client

#### — SSR과 세션 기반 요청을 위한 client

**의도**

- 사용자 세션 필요
- 요청마다 결과가 달라질 수 있음
- 정적 캐시 대상 ❌

**핵심 원칙**

- 쿠키 read / write ⭕
- 요청 컨텍스트 의존 ⭕
- 캐싱 ❌ (기본 `no-store`)

```
export const createServerSideClient = async (options) =>
  createServerClient(SUPABASE_URL, ANON_KEY, {
    global: {
      fetch: createFetch({
        cache: 'no-store',
        ...options,
      }),
    },
    cookies: { getAll, setAll },
  });
```

이 client는:

- 정적 캐시를 절대 기대하지 않고
- 대신 **세션 일관성**을 보장하는 역할을 맡는다.

------

### 7.4 Browser Client

#### — CSR과 사용자 UX를 위한 client

**의도**

- 클라이언트 사이드 상호작용
- 사용자 UX 중심
- 브라우저 생명주기 내 상태 유지

**핵심 원칙**

- CSR 전용
- 인스턴스 중복 생성 방지

```
let client;

export function getSupabaseBrowserClient() {
  if (!client) {
    client = createBrowserClient(SUPABASE_URL, ANON_KEY);
  }
  return client;
}
```

이 client는 서버 캐싱과는 무관하며,
 **UI와 사용자 경험**에만 집중한다.

------

### 7.5 Admin Client

#### — 서버 전용, 권한 우회 client

**의도**

- 관리자 전용 작업
- RLS 우회
- 일반 사용자 흐름과 완전히 분리

**핵심 원칙**

- service role key 사용
- 세션 / 쿠키 ❌

```
export const createAdminServerClient = () =>
  createClient(SUPABASE_URL, SERVICE_ROLE_KEY, {
    auth: {
      persistSession: false,
      autoRefreshToken: false,
    },
  });
```

------

### 7.6 분리 전략의 핵심 요약

| Client  | 실행 환경 | 쿠키 | 캐시     | 목적        |
| ------- | --------- | ---- | -------- | ----------- |
| Static  | SSG / ISR | ❌    | ⭕        | 정적 페이지 |
| Server  | SSR       | ⭕    | ❌        | 세션 기반   |
| Browser | CSR       | ⭕    | 브라우저 | UX          |
| Admin   | Server    | ❌    | ❌        | 권한 우회   |





### 기존 API 구조의 한계

기존 API 함수들은 대부분 이런 형태였다.

```
async function getBulletinById(id: string) {
  const supabase = await createServerSideClient();
  return supabase
    .from('bulletins')
    .select('*')
    .eq('id', id)
    .single();
}
```

이 구조의 특징은 단순하다.

- API 함수 내부에서
- Supabase client를 직접 생성하고
- 그 client에 강하게 결합되어 있다

이게 왜 문제가 되었을까?

------

### 8.2 같은 API, 다른 실행 환경

`getBulletinById(id)`라는 API는
 의미적으로는 하나의 쿼리다.

하지만 실행 환경에 따라 요구사항은 달랐다.

- **주보 상세 페이지**
  - 빌드 타임 / SSG
  - Static Client 필요
- **관리자 미리보기 페이지**
  - SSR
  - Server Client 필요

하지만 기존 구조에서는:

- API 내부에서 `createServerSideClient()`를 호출하고 있었기 때문에
- 이 API는 **Static 환경에서 절대 사용할 수 없었다**

즉,

> **“쿼리는 같지만, 실행 환경 때문에
>  API를 나눠야 하는 상황”**이 되어버렸다.

------

### 8.3 잘못된 해결 방향이 보이기 시작했다

이 시점에서 떠오를 수 있는 해결책은 두 가지였다.

#### 선택 1. API 함수 복제

```
getBulletinByIdStatic()
getBulletinByIdServer()
```

- 쿼리는 동일
- client만 다름

하지만 이 방식은:

- 중복을 구조적으로 강제하고
- API가 늘어날수록 유지보수가 어려워진다

------

#### 선택 2. API 내부에서 분기

```
async function getBulletinById(id, mode) {
  const supabase =
    mode === 'static'
      ? createStaticClient()
      : await createServerSideClient();
}
```

이 방식도 곧바로 문제가 드러났다.

- 호출부가 실행 환경을 알아야 하고
- API 시그니처가 오염되며
- 책임이 뒤틀린다

즉,

> **API가 “무엇을 조회하는지”뿐 아니라
>  “어디서 실행되는지”까지 알아야 하는 구조**

가 되어버린다.

------

### 8.4 문제의 본질 재정의

이 시점에서 문제를 다시 정의했다.

> ❌ API가 client를 선택하고 있다
>  ✅ API는 client를 몰라야 한다

API의 책임은 하나다.

> **“무엇을 조회하는가”**

그런데 기존 구조에서는
 이 책임 위에 이런 역할이 덧씌워져 있었다.

> **“어떤 실행 환경에서 호출되었는가”**

이 두 책임이 섞이는 순간,
 API는 재사용 불가능해진다.

------

### 8.5 Service Layer라는 결론

그래서 도달한 결론은 이거였다.

> **Supabase client 생성 책임을
>  API 함수 바깥으로 완전히 밀어낸다.**

즉,

- client는 외부에서 결정하고
- API(쿼리)는 client를 주입받아 동작하게 한다

이를 구조로 고정한 것이
 **Service Layer**였다.

------

### 8.6 Domain Service 패턴

```
export const bulletinService = (supabase) => ({
  fetchBulletinDetailById: async (id: string) => {
    const res = await supabase
      .from('bulletins')
      .select('*')
      .eq('id', id)
      .single();

    return handleResponse(res);
  },
});
```

이 구조의 핵심은 단순하다.

- Supabase client는 외부 주입
- 쿼리는 하나
- 실행 환경에 대한 조건 없음

------

### 8.7 Service Factory: 환경별 진입점

```
export const getStaticService = (options) =>
  rootService(createStaticClient(options));

export const getServerService = async (options) =>
  rootService(await createServerSideClient(options));
```

이제:

- 실행 환경에 맞는 client는 factory에서 결정되고
- API 로직은 오직 쿼리만 담당한다

호출부에서는 이렇게만 판단하면 된다.

```
// 정적 페이지
const service = getStaticService({ tags: ['bulletin'], revalidate: 3600 });
const bulletin = await service.bulletin.fetchBulletinDetailById(id);
```

------

### 8.8 이 구조가 해결한 것

이 구조로 다음 문제들이 동시에 해결됐다.

- 같은 쿼리를
  - Static 환경에서도
  - SSR 환경에서도
  - Admin 환경에서도
- **복제 없이 재사용 가능**

그리고 무엇보다 중요한 점은 이거였다.

> **Service Layer는 캐싱을 위해 도입한 구조가 아니다.
>  실행 환경 차이로 API 로직이 무너지는 걸 막기 위해 등장한 구조다.**
