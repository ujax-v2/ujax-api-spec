# @ujax/api-spec

ujax 프로젝트의 **OpenAPI 스펙**과 **TypeScript 타입**을 관리하는 패키지입니다.

백엔드의 REST Docs 테스트에서 자동 생성된 API 명세를 프론트엔드에서 타입으로 사용할 수 있습니다.

---

## 구조

```
ujax-api-spec/
├── openapi3.yaml   # OpenAPI 3.0 스펙 (백엔드에서 생성)
├── types.ts        # TypeScript 타입 (yaml에서 자동 생성)
├── package.json
└── README.md
```

---

## 프론트엔드 사용법

### 1. 설치

```bash
npm install github:ujax-v2/ujax-api-spec
```

### 2. 타입 import

```typescript
import type { components } from '@ujax/api-spec/types';

type UserResponse = components['schemas']['UserResponse'];
type UserUpdateRequest = components['schemas']['UserUpdateRequest'];
```

### 3. API 함수 작성

```typescript
// src/api/user.ts
import type { components } from '@ujax/api-spec/types';

type UserResponse = components['schemas']['UserResponse'];

export async function getMe(): Promise<UserResponse> {
  const res = await fetch('/api/v1/users/me');
  return res.json();
}
```

### 4. 업데이트

백엔드에서 API가 변경되면:

```bash
npm update @ujax/api-spec
```

---

## 백엔드 사용법

### 1. REST Docs 테스트 작성

```java
@Test
void getMe() throws Exception {
    mockMvc.perform(get("/api/v1/users/me"))
        .andDo(document("user-get-me",
            resource(ResourceSnippetParameters.builder()
                .tag("User")
                .summary("내 정보 조회")
                .responseSchema(Schema.schema("UserResponse"))
                .responseFields(
                    fieldWithPath("id").type(NUMBER).description("유저 ID"),
                    fieldWithPath("email").type(STRING).description("이메일")
                )
                .build()
            )
        ));
}
```

### 2. OpenAPI 스펙 생성

```bash
./gradlew openapi3
```

### 3. api-spec 패키지 업데이트

```bash
# yaml 복사
cp api-spec/openapi3.yaml ../ujax-api-spec/

# 타입 재생성
cd ../ujax-api-spec
npm run generate

# 커밋 & 푸시
git add .
git commit -m "feat: API 스펙 업데이트"
git push
```

---

## 스크립트

| 명령어 | 설명 |
|--------|------|
| `npm run generate` | openapi3.yaml → types.ts 변환 |

---

## API 문서 확인

### Swagger UI (로컬)

```bash
cd ujax-server/swagger
python3 -m http.server 8081
# http://localhost:8081/swagger-ui.html
```

### 서버 실행 후

```
http://localhost:8080/swagger/swagger-ui.html
```

---

## 워크플로우

```
┌─────────────────────────────────────────────────────────────────┐
│  백엔드 (ujax-server)                                            │
│  1. Controller + Service 구현                                    │
│  2. REST Docs 테스트 작성 (@Tag("restDocs"))                      │
│  3. ./gradlew openapi3                                          │
│  4. api-spec/openapi3.yaml 생성됨                                │
└──────────────────────────┬──────────────────────────────────────┘
                           │ 복사
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  ujax-api-spec                                                   │
│  1. openapi3.yaml 붙여넣기                                       │
│  2. npm run generate → types.ts 생성                            │
│  3. git push                                                     │
└──────────────────────────┬──────────────────────────────────────┘
                           │ npm update
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  프론트엔드 (ujax-front)                                         │
│  1. npm update @ujax/api-spec                                   │
│  2. types.ts에서 타입 import                                     │
│  3. VSCode 자동완성으로 API 사용                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 관련 저장소

- [ujax-server](https://github.com/ujax-v2/ujax-server) - 백엔드
- [ujax-front](https://github.com/ujax-v2/ujax-front) - 프론트엔드
