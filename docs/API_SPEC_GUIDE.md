# API 스펙 공유 가이드

> 백엔드와 프론트엔드가 API 스펙을 공유하는 방법에 대한 상세 가이드입니다.

---

## 1. 배경 및 철학

### 왜 이 방식을 사용하나요?

기존 방식의 문제점:
- Swagger 어노테이션을 코드에 직접 작성 → 코드가 지저분해짐
- API 문서를 수동으로 작성 → 실제 API와 문서가 달라질 수 있음
- 프론트엔드가 API 스펙을 확인하기 어려움 → Slack으로 "이 필드 뭐예요?" 질문

### 우리의 방식: "테스트가 곧 문서"

```
REST Docs 테스트 작성
       ↓
테스트 실행 시 실제 응답 캡처
       ↓
OpenAPI 스펙 자동 생성
       ↓
TypeScript 타입 자동 생성
       ↓
프론트엔드에서 타입 사용
```

**장점:**
- 테스트가 통과해야 문서가 생성됨 → 문서가 항상 진실
- 백엔드가 작성한 description이 프론트 IDE까지 전달됨
- 타입 안전성 확보 → 런타임 에러 감소

---

## 2. 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│  ujax-server (백엔드)                                            │
│                                                                  │
│  Controller → Service → Repository                               │
│       ↓                                                          │
│  REST Docs 테스트 (@Tag("restDocs"))                              │
│       ↓                                                          │
│  ./gradlew openapi3                                              │
│       ↓                                                          │
│  build/api-spec/openapi3.yaml                                    │
│       ↓ (자동 복사)                                               │
│  api-spec/openapi3.yaml                                          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           │ 수동 복사 & push
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  ujax-api-spec (공유 패키지)                                      │
│                                                                  │
│  openapi3.yaml ──npm run generate──→ types.ts                   │
│                                                                  │
│  GitHub: https://github.com/ujax-v2/ujax-api-spec               │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           │ npm install / update
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  ujax-front (프론트엔드)                                          │
│                                                                  │
│  import type { components } from '@ujax/api-spec/types'          │
│       ↓                                                          │
│  VSCode 자동완성 + 타입 체크                                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 백엔드 가이드

### 3.1 REST Docs 테스트 작성

#### 기본 구조

```java
@Tag("restDocs")  // 이 태그가 있어야 문서화됨
@WebMvcTest(UserController.class)
@AutoConfigureRestDocs
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @DisplayName("내 정보 조회 API")
    void getMe() throws Exception {
        // given
        User user = createTestUser(1L, "test@example.com", "테스트유저");
        given(userService.getUser(anyLong())).willReturn(user);

        // when & then
        mockMvc.perform(get("/api/v1/users/me")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andDo(document("user-get-me",  // 문서 식별자
                preprocessRequest(prettyPrint()),
                preprocessResponse(prettyPrint()),
                resource(ResourceSnippetParameters.builder()
                    .tag("User")                              // Swagger 태그
                    .summary("내 정보 조회")                    // API 요약
                    .description("로그인한 사용자의 정보를 조회합니다")  // 상세 설명
                    .responseSchema(Schema.schema("UserResponse"))     // 스키마 이름
                    .responseFields(
                        fieldWithPath("id").type(NUMBER).description("유저 ID"),
                        fieldWithPath("email").type(STRING).description("이메일"),
                        fieldWithPath("name").type(STRING).description("이름"),
                        fieldWithPath("profileImageUrl").type(STRING).description("프로필 이미지 URL").optional(),
                        fieldWithPath("provider").type(STRING).description("인증 제공자 (GOOGLE, KAKAO, LOCAL)")
                    )
                    .build()
                )
            ));
    }
}
```

#### Request Body가 있는 경우

```java
@Test
void updateMe() throws Exception {
    UserUpdateRequest request = new UserUpdateRequest("새이름", "https://example.com/profile.jpg");

    mockMvc.perform(patch("/api/v1/users/me")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(request)))
        .andExpect(status().isOk())
        .andDo(document("user-update-me",
            resource(ResourceSnippetParameters.builder()
                .tag("User")
                .summary("내 정보 수정")
                .requestSchema(Schema.schema("UserUpdateRequest"))   // 요청 스키마
                .responseSchema(Schema.schema("UserResponse"))       // 응답 스키마
                .requestFields(
                    fieldWithPath("name").type(STRING).description("수정할 이름 (30자 이내)").optional(),
                    fieldWithPath("profileImageUrl").type(STRING).description("수정할 프로필 이미지 URL").optional()
                )
                .responseFields(
                    fieldWithPath("id").type(NUMBER).description("유저 ID"),
                    // ... 나머지 필드
                )
                .build()
            )
        ));
}
```

#### Path Variable이 있는 경우

```java
@Test
void getUser() throws Exception {
    mockMvc.perform(get("/api/v1/users/{userId}", 1L))
        .andExpect(status().isOk())
        .andDo(document("user-get-by-id",
            resource(ResourceSnippetParameters.builder()
                .tag("User")
                .summary("특정 유저 조회")
                .pathParameters(
                    parameterWithName("userId").description("조회할 유저 ID")
                )
                .responseSchema(Schema.schema("UserResponse"))
                .responseFields(...)
                .build()
            )
        ));
}
```

#### Query Parameter가 있는 경우

```java
@Test
void searchUsers() throws Exception {
    mockMvc.perform(get("/api/v1/users")
            .param("keyword", "테스트")
            .param("page", "0")
            .param("size", "10"))
        .andExpect(status().isOk())
        .andDo(document("user-search",
            resource(ResourceSnippetParameters.builder()
                .tag("User")
                .summary("유저 검색")
                .queryParameters(
                    parameterWithName("keyword").description("검색 키워드"),
                    parameterWithName("page").description("페이지 번호 (0부터 시작)").optional(),
                    parameterWithName("size").description("페이지 크기").optional()
                )
                .responseSchema(Schema.schema("UserListResponse"))
                .responseFields(...)
                .build()
            )
        ));
}
```

### 3.2 OpenAPI 스펙 생성

```bash
# 테스트 실행 + OpenAPI 스펙 생성
./gradlew openapi3

# 결과물
# - build/api-spec/openapi3.yaml
# - api-spec/openapi3.yaml (자동 복사됨)
```

### 3.3 api-spec 패키지 업데이트

```bash
# 1. yaml 복사
cp api-spec/openapi3.yaml ../ujax-api-spec/

# 2. 타입 재생성
cd ../ujax-api-spec
npm run generate

# 3. 변경사항 확인
git diff

# 4. 커밋 & 푸시
git add .
git commit -m "feat: User API 스펙 추가"
git push
```

---

## 4. 프론트엔드 가이드

### 4.1 패키지 설치

```bash
# 최초 설치
npm install github:ujax-v2/ujax-api-spec

# 업데이트 (백엔드에서 API 변경 후)
npm update @ujax/api-spec
```

### 4.2 타입 사용

#### 기본 사용법

```typescript
// src/api/user.ts
import type { components } from '@ujax/api-spec/types';

// 스키마에서 타입 추출
type UserResponse = components['schemas']['UserResponse'];
type UserUpdateRequest = components['schemas']['UserUpdateRequest'];

// API 함수 작성
export async function getMe(): Promise<UserResponse> {
  const res = await fetch('/api/v1/users/me');
  if (!res.ok) throw new Error('Failed to fetch user');
  return res.json();
}

export async function updateMe(data: UserUpdateRequest): Promise<UserResponse> {
  const res = await fetch('/api/v1/users/me', {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!res.ok) throw new Error('Failed to update user');
  return res.json();
}
```

#### 컴포넌트에서 사용

```tsx
// src/components/Profile.tsx
import { useEffect, useState } from 'react';
import { getMe, updateMe } from '../api/user';
import type { components } from '@ujax/api-spec/types';

type UserResponse = components['schemas']['UserResponse'];

export function Profile() {
  const [user, setUser] = useState<UserResponse | null>(null);

  useEffect(() => {
    getMe().then(setUser);
  }, []);

  const handleUpdateName = async (name: string) => {
    const updated = await updateMe({ name });
    setUser(updated);
  };

  if (!user) return <div>Loading...</div>;

  return (
    <div>
      <p>이름: {user.name}</p>
      <p>이메일: {user.email}</p>
      <p>인증: {user.provider}</p>
    </div>
  );
}
```

### 4.3 VSCode 자동완성 활용

```typescript
// updateMe 호출 시
updateMe({
  // Ctrl+Space 누르면 자동완성
  name: '',           // string | null | undefined
  profileImageUrl: '' // string | null | undefined
});

// 마우스 hover 시 description 표시
// name?: string | null
// "수정할 이름 (30자 이내)"
```

---

## 5. 새 API 추가 체크리스트

### 백엔드

- [ ] Controller 메서드 구현
- [ ] Service 로직 구현
- [ ] REST Docs 테스트 작성 (`@Tag("restDocs")`)
- [ ] `./gradlew openapi3` 실행
- [ ] `api-spec/openapi3.yaml` 생성 확인
- [ ] `ujax-api-spec`에 yaml 복사
- [ ] `npm run generate` 실행
- [ ] `types.ts` 변경 확인
- [ ] git commit & push
- [ ] 프론트엔드에 알림

### 프론트엔드

- [ ] `npm update @ujax/api-spec` 실행
- [ ] 새 타입 import
- [ ] API 함수 작성
- [ ] 컴포넌트에서 사용

---

## 6. 트러블슈팅

### Q: types.ts에 새 스키마가 없어요

**원인**: `responseSchema()` 또는 `requestSchema()`를 지정하지 않음

**해결**:
```java
resource(ResourceSnippetParameters.builder()
    .responseSchema(Schema.schema("MyResponse"))  // 이거 추가
    .responseFields(...)
    .build()
)
```

### Q: 스키마 이름이 이상해요 (api-v1-users-me-123456)

**원인**: `Schema.schema()` 미지정

**해결**: 위와 동일

### Q: VSCode에서 타입이 안 보여요

**원인**: 패키지 설치 안 됨 또는 캐시 문제

**해결**:
```bash
npm update @ujax/api-spec
# VSCode 재시작 또는 Cmd+Shift+P → "TypeScript: Restart TS Server"
```

### Q: openapi3.yaml이 생성되지 않아요

**원인**: 테스트에 `@Tag("restDocs")`가 없음

**해결**: 테스트 클래스에 태그 추가
```java
@Tag("restDocs")
class MyControllerTest { ... }
```

---

## 7. 관련 링크

| 항목 | URL |
|------|-----|
| ujax-api-spec | https://github.com/ujax-v2/ujax-api-spec |
| ujax-server | https://github.com/ujax-v2/ujax-server |
| ujax-front | https://github.com/ujax-v2/ujax-front |
| Swagger UI (로컬) | http://localhost:8081/swagger-ui.html |
| openapi-typescript | https://openapi-ts.dev/ |
| restdocs-api-spec | https://github.com/ePages-de/restdocs-api-spec |
