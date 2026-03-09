✅ 완료

# Phase 7: Phase 5 infra 렌더링 loading/error 변수명 수정

## 목표

`data-state="field.loading"` / `data-state="field.error"` 조건에서 생성되는 변수명을 useQuery hook에서 정의한 변수명과 일치시킨다.

## 현상

useQuery hook은 `{toLowerFirst(operationId)}Data` 접미사 패턴으로 변수를 정의한다:

```tsx
const { data: listCoursesData, isLoading: listCoursesDataLoading, error: listCoursesDataError } = useQuery(...)
```

그러나 `renderStateJSX()`에서 `.loading`/`.error` suffix를 처리할 때, operationId가 아닌 field 이름만으로 변수명을 만든다:

```go
// react_templates.go:401-406
case strings.HasSuffix(s.Condition, ".loading"):
    field := strings.TrimSuffix(s.Condition, ".loading")
    cond = toLowerFirst(field) + "Loading"   // → "coursesLoading" (정의되지 않음)
```

## 원인

`renderStateJSX()`는 부모 FetchBlock의 operationId를 알지 못한다. `.loading`/`.error` condition의 field 이름(예: `courses`)만으로 변수명을 생성하므로, useQuery가 정의한 `listCoursesDataLoading`과 불일치한다.

## 변경 방안

`renderStateJSX`에서 `.loading`/`.error` condition 처리 시, 변수명 패턴을 useQuery와 동일하게 맞춘다.

**문제:** `renderStateJSX`는 현재 operationId를 모른다. condition 문자열 `"courses.loading"`에서 operationId `"ListCourses"`를 역추적할 방법이 없다.

**해결:** `renderStateJSX`가 호출되는 컨텍스트에서 부모 fetch의 alias(`toLowerFirst(operationId) + "Data"`)가 `dataVar`로 이미 전달된다. `.loading`/`.error` 변수명도 이 `dataVar`를 기반으로 생성한다.

```go
// 수정 전
case strings.HasSuffix(s.Condition, ".loading"):
    field := strings.TrimSuffix(s.Condition, ".loading")
    cond = toLowerFirst(field) + "Loading"

// 수정 후
case strings.HasSuffix(s.Condition, ".loading"):
    cond = dataVar + "Loading"

case strings.HasSuffix(s.Condition, ".error"):
    cond = dataVar + "Error"
```

`dataVar`는 `renderChildNodes` → `renderStateJSX` 호출 체인에서 부모 fetch의 alias가 전달되므로, `listCoursesData` → `listCoursesDataLoading`이 된다.

**검증:** `dataVar`가 빈 문자열인 경우(fetch 컨텍스트 밖)에는 기존 로직 유지가 필요한지 확인한다. 현재 `.loading`/`.error` state는 fetch 안에서만 의미가 있으므로 dataVar가 항상 존재한다.

## 변경 파일 목록

| 파일 | 변경 유형 | 내용 |
|---|---|---|
| `generator/react_templates.go` | 수정 | `renderStateJSX()`의 `.loading`/`.error` 변수명을 `dataVar` 기반으로 변경 |
| `generator/generator_test.go` | 수정 | `.loading`/`.error` state 변수명 일치 테스트 추가 |

## 의존성

없음.

## 검증 방법

1. 기존 테스트 전체 통과
2. 새 테스트: fetch 안에 `data-state="field.loading"` 선언 시 생성 코드에서 `{operationId}DataLoading` 변수명 사용 확인
3. `go test ./... -count=1`
