---
layout: single
title: 주보 페이지로 설계한 Next.js 렌더링 및 캐싱 전략
category: TIL
toc: true
toc_sticky: true
---

# Supabase client 분리와 Service Layer 도입기

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

