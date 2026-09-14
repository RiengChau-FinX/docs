# Task Brief — Xây dựng luồng Đăng nhập Web (`/v1/auth/web-login`)

| | |
|---|---|
| **Mã task** | WEB-LOGIN |
| **Repo chính** | `authentication-service` |
| **Ngày lập** | 2026-09-14 |
| **Tài liệu thiết kế** | [`docs/web-login-design.md`](web-login-design.md) — đọc **trước** khi bắt đầu |
| **Đặc tả API** | [`docs/web-login-api.md`](web-login-api.md) — hợp đồng chính thức với FE |
| **Nguồn yêu cầu** | `prompts/002-login-for-web.md`, mockup `prompts/images/Login.png` |

---

## 0. Tóm tắt việc phải làm

Thêm luồng đăng nhập dành riêng cho Vikki Web gồm 3 lớp: **CAPTCHA tự implement (Redis)** → **số điện thoại + mật khẩu** → **SMS OTP**. Token do Cognito phát hành qua một **app client riêng cho web** (dùng chung user pool với mobile), hết hạn **30 phút**.

**4 API mới** (chi tiết ở [`web-login-api.md`](web-login-api.md)):

| | Method | Path |
|---|---|---|
| N1 | `GET` | `/api/authentication/v1/auth/captcha` |
| N2 | `POST` | `/api/authentication/v1/auth/web-login` |
| N3 | `POST` | `/api/authentication/v1/auth/web-login/resend-otp` |
| N4 | `POST` | `/api/authentication/v1/auth/web-login/verify-otp` |

---

## 1. Nguyên tắc bắt buộc (vi phạm = reject PR)

| # | Nguyên tắc | Lý do |
|---|---|---|
| NT1 | **Không sửa bất kỳ hành vi nào của luồng mobile** — không đổi chữ ký method đang dùng, không đổi cấu hình Cognito hiện có | Mobile đang chạy PROD |
| NT2 | Toàn bộ tính năng nằm sau feature flag `WEB_LOGIN_ENABLED`, **mặc định `false`** | Bật/tắt không cần deploy lại |
| NT3 | **Không lưu mật khẩu** ở bất kỳ đâu (log, Redis, DB) | Chuẩn bảo mật |
| NT4 | Token Cognito trong Redis **phải mã hoá AES-GCM**, xoá ngay sau khi dùng | ADR-001, mục A2/A4 |
| NT5 | **Không** log mật khẩu, OTP, đáp án CAPTCHA, token (kể cả mức `DEBUG`) | ADR-001, mục A7 |
| NT6 | Controller mới **phải** map đúng `/v1/auth`, **không** thêm endpoint vào `AuthController` | `AuthController` map cả `/v1/auth` và `/v2/auth` ⇒ sẽ rò endpoint sang `/v2` |
| NT7 | CAPTCHA dùng **font nhúng trong resource**, không dùng font hệ thống | Base image `liberica-runtime-container:*-slim-glibc` có thể không có font |

---

## 2. Bảng tiến độ (checklist)

| Bước | Tên | Người làm | Phụ thuộc | Xong |
|---|---|---|---|---|
| S0 | Chuẩn bị hạ tầng Cognito & secret | Infra/Security | — | ☐ |
| S1 | Cấu hình & enum kênh | BE | S0 | ☐ |
| S2 | `CognitoUserService` chọn client theo kênh | BE | S1 | ☐ |
| S3 | Bổ sung mã lỗi | BE | — | ☐ |
| S4 | `WebSessionTokenCipher` (AES-GCM) | BE | S1 | ☐ |
| S5 | `CaptchaService` + font nhúng | BE | S1, S3 | ☐ |
| S6 | Model & repository phiên web trên Redis | BE | S1, S4 | ☐ |
| S7 | `WebLoginService` — bước 1 (N2) | BE | S2, S5, S6 | ☐ |
| S8 | Gửi lại OTP (N3) | BE | S7 | ☐ |
| S9 | Xác minh OTP & cấp token (N4) | BE | S7 | ☐ |
| S10 | Controller + DTO + validation | BE | S5, S7, S8, S9 | ☐ |
| S11 | Rate limit theo IP | BE | S10 | ☐ |
| S12 | Kafka audit event `channel=WEB` | BE | S9 | ☐ |
| S13 | Unit test | BE | S2–S12 | ☐ |
| S14 | Integration test (Testcontainers) | BE | S13 | ☐ |
| S15 | Helm: env + secret | BE + DevOps | S0, S10 | ☐ |
| S16 | Khai route API Gateway + authorizer | Infra | S10, S15 | ☐ |
| S17 | Deploy SIT + E2E | BE + QA | S15, S16 | ☐ |
| S18 | Bật PROD | BE + DevOps | S17 | ☐ |

**Có thể làm song song:** S3, S4, S5 độc lập nhau. S13 viết dần theo từng bước, không dồn cuối.

---

## 3. Chi tiết từng bước

### S0 — Chuẩn bị hạ tầng Cognito & secret
> **Người làm:** Infra/Security · **Chặn:** S1, S15 · **Môi trường:** SIT → UAT → PROD

**Việc cụ thể**

1. Tạo **user-pool app client cho web**, dùng **chung user pool** với mobile (`AWS_USER_POOL_ID`; PROD = `ap-southeast-1_DnBLrQVyE`):
   ```bash
   aws cognito-idp create-user-pool-client \
     --region ap-southeast-1 \
     --user-pool-id "$USER_POOL_ID" \
     --client-name "vikki-web-app" \
     --generate-secret \
     --explicit-auth-flows ALLOW_ADMIN_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH \
     --access-token-validity 30 --id-token-validity 30 --refresh-token-validity 1 \
     --token-validity-units AccessToken=minutes,IdToken=minutes,RefreshToken=days \
     --prevent-user-existence-errors ENABLED \
     --enable-token-revocation
   ```
   ⚠️ **Bắt buộc `--generate-secret`** — code tính `SECRET_HASH` bằng HMAC-SHA256 với client secret; thiếu secret thì Cognito trả `NotAuthorizedException`, biểu hiện **giống hệt sai mật khẩu**.
2. Ghi lại `ClientId` + `ClientSecret` ngay (secret không xem lại được).
3. Sinh **khoá AES-256** cho `WEB_LOGIN_TOKEN_ENC_KEY` và **pepper** cho CAPTCHA, lưu vào AWS Secrets Manager.
4. Tạo 3 key mới trong K8s secret `authentication-service-secret`: `web_app_client_secret`, `web_login_token_enc_key`, `captcha_pepper`.
5. Chốt cơ chế pre-login cho web (M2M client + scope riêng, hay dùng chung mobile).

**Definition of Done**
- ☐ `describe-user-pool-client` trả về `AccessTokenValidity=30`, `TokenValidityUnits.AccessToken=minutes`
- ☐ 3 key secret đã tồn tại ở cả 3 môi trường
- ☐ Đã trả lời: 4 route login dùng authorizer nào

---

### S1 — Cấu hình & enum kênh
> **Người làm:** BE · **Phụ thuộc:** S0

**Việc cụ thể**

1. `infra/.../configuration/AwsConfig.java` — thêm vào class `Cognito`:
   ```java
   private String webAppClientId;
   private String webAppClientSecret;
   ```
2. `api/src/main/resources/application.yml`:
   ```yaml
   aws:
     cognito:
       web-app-client-id: ${AWS_WEB_CLIENT_ID:}
       web-app-client-secret: ${AWS_WEB_CLIENT_SECRET:}

   web-login:
     enabled: ${WEB_LOGIN_ENABLED:false}
     session-ttl: ${WEB_LOGIN_SESSION_TTL:PT5M}
     otp-variant: ${WEB_LOGIN_OTP_VARIANT:OTP_AUTH}
     resend-cooldown: ${WEB_LOGIN_OTP_RESEND_COOLDOWN:PT30S}
     min-usable-token-lifetime: ${WEB_LOGIN_MIN_TOKEN_LIFETIME:PT10M}
     token-encryption-key: ${WEB_LOGIN_TOKEN_ENC_KEY:}

   captcha:
     length: ${CAPTCHA_LENGTH:6}
     ttl: ${CAPTCHA_TTL:PT2M}
     max-attempts: ${CAPTCHA_MAX_ATTEMPTS:3}
     issue-rate-per-ip-per-minute: ${CAPTCHA_ISSUE_RATE:20}
     image: { width: ${CAPTCHA_IMAGE_WIDTH:160}, height: ${CAPTCHA_IMAGE_HEIGHT:60} }
     pepper: ${CAPTCHA_PEPPER:}
   ```
3. Tạo `core/.../model/enums/AuthChannel.java`: `MOBILE`, `WEB`.

**Definition of Done**
- ☐ App khởi động được khi các biến mới **để trống** (không được fail startup khi flag tắt)
- ☐ `@ConfigurationProperties` bind đúng, có test bind config

---

### S2 — `CognitoUserService` chọn client theo kênh
> **Người làm:** BE · **Phụ thuộc:** S1 · **⚠️ Bước rủi ro cao nhất với mobile**

**Việc cụ thể**

1. Thêm method nạp chồng vào interface `CognitoUserService`:
   ```java
   Either<ErrorCode, LoginResult> verifyPasswordByCif(String u, String p, AuthChannel channel);
   ```
2. `CognitoUserServiceImpl`:
   - Method **cũ** giữ nguyên chữ ký, gọi `verifyPasswordByCif(u, p, AuthChannel.MOBILE)` ⇒ hành vi mobile **không đổi**.
   - **Tách tham số secret** cho `calculateSecretHash(username, clientSecret)` — hiện đang hardcode `appClientSecret`.
   - `buildAuthRequest(...)` nhận thêm `clientId`.
3. `refreshToken(...)` cũng phải chọn client theo kênh — lưu `channel` vào phiên/claim để biết đường refresh.

**Definition of Done**
- ☐ Test: gọi bản cũ ⇒ dùng `appClientId` + `appClientSecret` (không đổi hành vi)
- ☐ Test: gọi với `WEB` ⇒ dùng `webAppClientId` + `webAppClientSecret`
- ☐ Test: `SECRET_HASH` của 2 kênh **khác nhau**
- ☐ Toàn bộ test cũ của `CognitoUserServiceImpl` vẫn xanh

---

### S3 — Bổ sung mã lỗi
> **Người làm:** BE · **Phụ thuộc:** không

Thêm vào `core/.../businessexception/ErrorCode.java` (giữ đúng khuôn `TÊN("MÃ", "message", HttpStatus)`):

| Enum | Mã | HTTP |
|---|---|---|
| `CAPTCHA_INVALID` | `CAPTCHA_INVALID` | 400 |
| `CAPTCHA_EXPIRED` | `CAPTCHA_EXPIRED` | 400 |
| `CAPTCHA_RATE_LIMITED` | `CAPTCHA_RATE_LIMITED` | 429 |
| `WEB_SESSION_EXPIRED` | `SESSION_EXPIRED` | 400 |
| `OTP_RESEND_TOO_SOON` | `OTP_RESEND_TOO_SOON` | 429 |
| `OTP_GENERATION_LIMIT_EXCEEDED` | `OTP_GENERATION_LIMIT_EXCEEDED` | 429 |
| `OTP_VERIFICATION_LIMIT_EXCEEDED` | `OTP_VERIFICATION_LIMIT_EXCEEDED` | 429 |

**Definition of Done**
- ☐ Mã lỗi khớp **từng ký tự** với [`web-login-api.md`](web-login-api.md)
- ☐ Không trùng mã với enum đang có

---

### S4 — `WebSessionTokenCipher` (AES-GCM)
> **Người làm:** BE · **Phụ thuộc:** S1

**Việc cụ thể**

1. Tạo `infra/.../crypto/WebSessionTokenCipher.java`:
   - `AES/GCM/NoPadding`, khoá 256-bit từ `web-login.token-encryption-key`
   - **IV 12 byte ngẫu nhiên mỗi lần mã hoá**, ghép `IV || ciphertext || tag` rồi Base64
   - `encrypt(String plain)` / `decrypt(String encoded)`
2. Giải mã thất bại ⇒ ném lỗi riêng, **không** trộn lẫn với lỗi nghiệp vụ (để phân biệt sự cố xoay khoá).

**Definition of Done**
- ☐ Round-trip trả đúng chuỗi gốc
- ☐ Mã hoá cùng một chuỗi 2 lần cho ra 2 kết quả **khác nhau**
- ☐ Giải mã bằng khoá khác ⇒ ném lỗi, không trả rác

---

### S5 — `CaptchaService` + font nhúng
> **Người làm:** BE · **Phụ thuộc:** S1, S3

**Việc cụ thể**

1. Thêm `api/src/main/resources/fonts/DejaVuSans.ttf` (hoặc font tương đương, license cho phép nhúng).
2. `core/.../service/CaptchaService.java` + `infra/.../service/impl/CaptchaServiceImpl.java`:
   - Sinh 6 ký tự từ bảng **đã loại** `0 O o 1 l I`, dùng `SecureRandom`
   - Render PNG 160×60 bằng Java2D: nền `#0F2A4A`, chữ `#5AC8FA`, xoay ±14°, 4–6 đường nhiễu, ~200 chấm
   - Nạp font bằng `Font.createFont(Font.TRUETYPE_FONT, resourceStream)` **một lần** lúc khởi động
   - Lưu Redis `auth:web:captcha:{captchaId}` TTL 120s: `{answerHash, attempts, bindHash, issuedAt}`
     - `answerHash = SHA-256(lower(answer) + ":" + captchaId + ":" + pepper)`
     - `bindHash = SHA-256(clientIp + "|" + userAgent)`
   - `verify(captchaId, answer)`: so sánh **timing-safe** (`MessageDigest.isEqual`), **`DEL` key dù đúng hay sai**
   - `issue(previousCaptchaId)`: nếu có thì `DEL` mã cũ trước
   - Rate limit phát hành: `auth:web:captcha:rate:{ipHash}` — `INCR` + `EXPIRE 60`, ngưỡng 20

**Definition of Done**
- ☐ Ảnh trả về là PNG hợp lệ, decode được
- ☐ So khớp **không phân biệt hoa/thường** (ảnh `A3eR7p`, nhập `a3er7p` ⇒ đúng)
- ☐ Dùng lại `captchaId` lần 2 ⇒ `CAPTCHA_EXPIRED`
- ☐ Re-render ⇒ mã cũ hết hiệu lực ngay
- ☐ Redis **không** chứa đáp án dạng chữ thường đọc được

---

### S6 — Model & repository phiên web trên Redis
> **Người làm:** BE · **Phụ thuộc:** S1, S4

**Việc cụ thể**

1. `core/.../model/memcache/WebLoginSession.java`:
   ```
   sessionId, cifNumber, phoneNumber, tokenEnc,
   tokenIssuedAt, originalExpiresIn, state, createdAt, resendCount
   ```
2. Repository thao tác Redis: `save(session, ttl)`, `find(sessionId)`, `delete(sessionId)`.
3. Key `auth:web:login:{sessionId}`, TTL lấy từ `web-login.session-ttl` (mặc định 5 phút).
4. `sessionId` = UUID v4 sinh bằng `SecureRandom`.

**Definition of Done**
- ☐ TTL đúng 300s, hết hạn thì `find` trả rỗng
- ☐ Đọc thẳng giá trị trong Redis **không** thấy chuỗi `eyJ` (JWT thô)

---

### S7 — `WebLoginService` bước 1 (API N2)
> **Người làm:** BE · **Phụ thuộc:** S2, S5, S6

**Thứ tự xử lý — bắt buộc đúng thứ tự này (fail-fast):**

| # | Việc | Lỗi trả |
|---|---|---|
| 1 | Verify CAPTCHA (xoá key ngay) | `CAPTCHA_INVALID` / `CAPTCHA_EXPIRED` |
| 2 | `partyService.verifyPhoneNumberLoginNewDevice(fullPhone)` | `PHONE_NUMBER_NOT_FOUND` |
| 3 | Kiểm tra trạng thái KH + hạn chế OFAC | `CUSTOMER_RESTRICTED_COUNTER_ONLY`, … |
| 4 | `authenticationService.checkLockedStatusAndVerifyPassword(cif, password)` với kênh `WEB` | `NOT_ALLOW_LOGIN_IN_LOCKED_TIME`, `FAILED_AUTHENTICATION`, `USER_PASSWORD_EXPIRED` |
| 5 | Mã hoá token + lưu phiên Redis (`state = OTP_PENDING`) | `SERVER_ERROR` |
| 6 | `otpService` → mfa-service `/v1/internal/otp/generate` | `OTP_GENERATION_LIMIT_EXCEEDED` |

> **CAPTCHA phải verify trước party lookup** — để một request bot không kích hoạt bất kỳ lời gọi downstream nào.

**Payload gửi mfa-service** (`sessionId = cifNumber`, `sessionKey = webSessionId` — giữ đúng khuôn mobile):
```json
{ "cifNumber": "<cif>", "otpVariant": "OTP_AUTH", "sessionId": "<cif>",
  "sessionKey": "<webSessionId>", "phoneNumber": "<+84...>", "data": { "channel": "WEB" } }
```
⚠️ `sessionId` tham gia sinh/kiểm OTP — bước S9 phải truyền lại **đúng** cặp `sessionId`/`sessionKey` này.

**Definition of Done**
- ☐ Response **không** chứa bất kỳ token nào
- ☐ CAPTCHA sai ⇒ **không** có lời gọi nào tới party-service (verify bằng mock)
- ☐ Mật khẩu sai ⇒ bộ đếm `authentication_status` tăng, dùng chung với mobile
- ☐ Gửi OTP lỗi ⇒ phiên Redis được dọn, không để rác

---

### S8 — Gửi lại OTP (API N3)
> **Người làm:** BE · **Phụ thuộc:** S7

1. Đọc phiên; không có ⇒ `SESSION_EXPIRED`.
2. Cooldown: `SET NX auth:web:otp:cooldown:{sessionId} EX 30`; đặt không được ⇒ `OTP_RESEND_TOO_SOON`.
3. Gọi lại `/v1/internal/otp/generate` với **đúng** `sessionId`/`sessionKey` cũ.
4. Tăng `resendCount`, **không gia hạn TTL phiên**.

**Definition of Done**
- ☐ Gọi 2 lần liên tiếp ⇒ lần 2 trả `OTP_RESEND_TOO_SOON`
- ☐ Quá 5 lần/giờ ⇒ mfa-service chặn, map ra `OTP_GENERATION_LIMIT_EXCEEDED`
- ☐ TTL phiên **không** bị kéo dài sau khi gửi lại

---

### S9 — Xác minh OTP & cấp token (API N4)
> **Người làm:** BE · **Phụ thuộc:** S7

1. Đọc phiên; không có ⇒ `SESSION_EXPIRED`.
2. Gọi mfa-service `/v1/internal/otp/verify`; sai ⇒ `INVALID_CHALLENGE_OTP` và **giữ nguyên phiên** (mockup cho phép thử lại).
3. Đúng ⇒ giải mã token, tính thời hạn còn lại:
   ```java
   long elapsed   = Duration.between(session.getTokenIssuedAt(), Instant.now()).toSeconds();
   long remaining = session.getOriginalExpiresIn() - elapsed;
   if (remaining < minUsableTokenLifetimeSeconds) { delete(session); return SESSION_EXPIRED; }
   ```
4. `DEL` key phiên **ngay** rồi mới trả token.
5. Trả `expiresIn = remaining` (**không** phải 1800).

**Definition of Done**
- ☐ `expiresIn` trả về **nhỏ hơn** 1800 đúng bằng thời gian đã trôi
- ☐ Gọi lại N4 với cùng `sessionId` ⇒ `SESSION_EXPIRED`
- ☐ OTP sai 1 lần ⇒ vẫn nhập lại được bằng cùng `sessionId`

---

### S10 — Controller + DTO + validation
> **Người làm:** BE · **Phụ thuộc:** S5, S7, S8, S9

1. Tạo **controller mới** `api/.../controller/web/WebAuthController.java`, `@RequestMapping("/v1/auth")` — **một path duy nhất** (xem NT6).
2. DTO: `WebLoginRequest`, `ResendWebOtpRequest`, `VerifyWebOtpRequest` + response tương ứng.
3. Validation: `@ValidPhoneNumber` (tái dùng), `@NotBlank` cho các trường bắt buộc. **Không** gắn `@RateLimitedRequestV3` (đó là reCAPTCHA của mobile).
4. Toàn bộ endpoint bọc bởi feature flag `web-login.enabled`; tắt ⇒ trả 404.
5. Trả lỗi qua `ExceptionAdviceHandler` sẵn có.

**Definition of Done**
- ☐ `GET /v2/auth/captcha` và `POST /v2/auth/web-login` trả **404** (không rò sang prefix khác)
- ☐ `WEB_LOGIN_ENABLED=false` ⇒ cả 4 endpoint trả 404
- ☐ Response khớp **đúng** schema trong [`web-login-api.md`](web-login-api.md)

---

### S11 — Rate limit theo IP
> **Người làm:** BE · **Phụ thuộc:** S10

1. Tạo `api/.../config/ClientIpKeyGenerator.java` — lấy IP client thật từ `X-Forwarded-For` (trusted proxy).
2. Thêm policy vào `application.yml`:
   ```yaml
   rate-limit:
     key-generators:
       - name: clientIpFromHeader
         generator: com.finx.authentication.api.config.ClientIpKeyGenerator
         params: ["X-Forwarded-For"]
     policies:
       - duration: ${WEB_LOGIN_RATE_LIMIT_DURATION:5m}
         count: ${WEB_LOGIN_RATE_LIMIT_COUNT:10}
         key-generator: clientIpFromHeader
         routes:
           - uri: ${server.servlet.context-path}/v1/auth/web-login
             method: POST
         block:
           duration: ${WEB_LOGIN_RATE_LIMIT_BLOCK_DURATION:15m}
   ```

⚠️ Sau ALB/CloudFront, `X-Forwarded-For` chứa **nhiều IP**. Lấy sai ⇒ toàn bộ traffic gom vào một key và **chặn nhầm hàng loạt**. Phải test sau ALB ở SIT.

**Definition of Done**
- ☐ Gọi 11 lần/5 phút từ một IP ⇒ lần 11 bị chặn
- ☐ Hai IP khác nhau ⇒ 2 quota độc lập
- ☐ Policy của mobile (`/v*/auth/login`, `/v*/auth/verify-phone-number`) **không đổi**

---

### S12 — Kafka audit event `channel=WEB`
> **Người làm:** BE · **Phụ thuộc:** S9

Sau khi cấp token thành công, publish auth audit event qua `KafkaEventService` sẵn có, thêm trường `channel = "WEB"`. Luồng web **không** ghi `device_info` (web không có `deviceId` vật lý).

**Definition of Done**
- ☐ Event lên đúng topic hiện hành, phân biệt được kênh
- ☐ Bảng `device_info` **không** phát sinh bản ghi mới từ luồng web

---

### S13 — Unit test
> **Người làm:** BE · **Phụ thuộc:** S2–S12

| Lớp | Ca bắt buộc |
|---|---|
| `CaptchaServiceImpl` | độ dài; không phân biệt hoa/thường; hết hạn; dùng lại; re-render huỷ mã cũ; vượt rate limit IP |
| `WebSessionTokenCipher` | round-trip; 2 lần mã hoá khác nhau; sai khoá ⇒ lỗi |
| `WebLoginServiceImpl` | thứ tự fail-fast; token không lọt vào response N2; `expiresIn` = còn lại; `< 600s` ⇒ `SESSION_EXPIRED`; gọi N4 lần 2 ⇒ `SESSION_EXPIRED` |
| `CognitoUserServiceImpl` | `MOBILE` dùng client cũ; `WEB` dùng client mới; SECRET_HASH đúng secret |

**Definition of Done** — ☐ Coverage phần code mới ≥ ngưỡng JaCoCo của repo · ☐ Toàn bộ test cũ vẫn xanh

---

### S14 — Integration test (Testcontainers)
> **Người làm:** BE · **Phụ thuộc:** S13

- Redis thật: TTL captcha, TTL phiên, cooldown resend
- WireMock cho party-service / Cognito / mfa-service
- Kiểm tra `/v2/auth/web-login` trả **404**
- Đọc thẳng key Redis: **không** chứa `eyJ`
- Sinh ảnh CAPTCHA **chạy trong container** (chặn hồi quy font)

---

### S15 — Helm: env + secret
> **Người làm:** BE + DevOps · **Phụ thuộc:** S0, S10

Thêm vào `prod-application-workload/workload/<env>/authentication-service/values.yaml`:

```yaml
      - name: AWS_WEB_CLIENT_ID
        value: "<web-client-id>"
      - name: AWS_WEB_CLIENT_SECRET
        valueFrom: { secretKeyRef: { name: authentication-service-secret, key: web_app_client_secret } }
      - name: WEB_LOGIN_ENABLED
        value: "false"        # bật ở S17/S18
      - name: WEB_LOGIN_TOKEN_ENC_KEY
        valueFrom: { secretKeyRef: { name: authentication-service-secret, key: web_login_token_enc_key } }
      - name: CAPTCHA_PEPPER
        valueFrom: { secretKeyRef: { name: authentication-service-secret, key: captcha_pepper } }
```

⚠️ 3 key secret phải tồn tại **trước** khi deploy, nếu không pod `CreateContainerConfigError`.

**Definition of Done** — ☐ Pod SIT khởi động xanh với flag tắt

---

### S16 — Khai route API Gateway + authorizer
> **Người làm:** Infra · **Phụ thuộc:** S10, S15

1. Khai 4 route (path đầy đủ ở [`web-login-api.md`](web-login-api.md)) trên API Gateway phục vụ Vikki Web, `x-api-id` **chưa dùng**, integration trỏ ALB nội bộ của `authentication-service`.
2. **Bổ sung `client_id` của web app client vào danh sách chấp nhận của Lambda authorizer** — nếu thiếu, đăng nhập thành công nhưng **mọi API nghiệp vụ trả 401**.
3. Bật **CORS** cho 4 route (web gọi từ origin khác; mobile không cần nên hiện chưa có).

> ❗ **Cần xác nhận trước khi làm:** Vikki Web đi qua API Gateway **nào**. Nếu web dùng gateway riêng (không phải gateway `prod` của mobile) thì route khai ở kho cấu hình của gateway đó, **không** khai vào `prod-openapi-configs/prod/internal-apis.yaml`. Hỏi Infra chốt trước.

**Definition of Done**
- ☐ Gọi được N1 từ ngoài internet ở SIT
- ☐ Token web gọi được **ít nhất 1 API nghiệp vụ** (không phải 401) — **bài test chặn rủi ro lớn nhất**

---

### S17 — Deploy SIT + E2E
> **Người làm:** BE + QA · **Phụ thuộc:** S15, S16

| # | Kịch bản | Kỳ vọng |
|---|---|---|
| E1 | Luồng thành công đầy đủ | Nhận JWT; `expiresIn` < 1800 đúng bằng thời gian đã trôi |
| E2 | Bấm ↻ 5 lần rồi nhập mã cuối | Thành công; 4 mã trước đều fail |
| E3 | Sai mật khẩu 5 lần | Tài khoản khoá; mobile cũng khoá |
| E4 | Sai OTP 5 lần | mfa-service khoá 1 giờ |
| E5 | Gửi lại mã 2 lần liên tiếp | Lần 2 `OTP_RESEND_TOO_SOON` |
| E6 | Chờ 6 phút rồi nhập OTP | `SESSION_EXPIRED` |
| E7 | Dùng lại `sessionId` sau khi đã lấy token | `SESSION_EXPIRED` |
| E8 | Đăng nhập web và mobile song song | 2 phiên độc lập, token khác `client_id` |
| E9 | Token web sau 30 phút | Hết hạn; `refresh-token` cấp lại được |
| E10 | **Dùng token web gọi API nghiệp vụ thật** | **200**, không phải 401 |

**Kiểm thử bảo mật bắt buộc**
- ☐ Bỏ qua captcha (`captchaId` rỗng/bịa) ⇒ fail
- ☐ Giải captcha ở IP A, dùng ở IP B ⇒ fail
- ☐ Dò OTP bằng script ⇒ khoá sau 5 lần
- ☐ Log **không** chứa mật khẩu / OTP / token / đáp án captcha
- ☐ Chạy `/security-review` trên nhánh trước khi merge

---

### S18 — Bật PROD
> **Người làm:** BE + DevOps · **Phụ thuộc:** S17

1. Deploy PROD với `WEB_LOGIN_ENABLED=false`.
2. Bật flag ngoài giờ cao điểm.
3. Theo dõi **30 phút đầu**: tỉ lệ lỗi 4xx/5xx của 4 endpoint, tỉ lệ giải captcha thành công, số SMS OTP gửi đi, `used_memory` của Redis, tỉ lệ 401 ở API nghiệp vụ.
4. **Rollback:** đặt `WEB_LOGIN_ENABLED=false` — không cần deploy lại.

**Definition of Done**
- ☐ Có runbook rollback
- ☐ Dashboard/alert cho 5 chỉ số trên
- ☐ Luồng mobile **không** thay đổi chỉ số nào

---

## 4. Định nghĩa hoàn thành toàn task

- ☐ 4 API chạy đúng [`web-login-api.md`](web-login-api.md) trên PROD sau feature flag
- ☐ Luồng mobile không thay đổi hành vi — test cũ xanh, chỉ số PROD không đổi
- ☐ Coverage đạt ngưỡng JaCoCo, `/security-review` sạch
- ☐ Token web hết hạn đúng 30 phút; refresh hoạt động
- ☐ Token web gọi được API nghiệp vụ (E10 pass)
- ☐ Runbook rollback + dashboard đã bàn giao vận hành

---

## 5. Việc cần chốt trước khi bắt đầu

| # | Câu hỏi | Người quyết | Chặn bước |
|---|---|---|---|
| Q1 | Vikki Web đi qua API Gateway nào? Route khai ở kho nào? | Infra | S16 |
| Q2 | 4 route login dùng authorizer nào (pre-login / API key / public)? | Infra + Security | S0, S16 |
| Q3 | Có gộp `PHONE_NUMBER_NOT_FOUND` vào `FAILED_AUTHENTICATION` để chống dò tài khoản? | Security + UX | S7 |
| Q4 | Mockup hiện **số điện thoại đầy đủ** — Compliance có yêu cầu che không? | Compliance | S7 |
| Q5 | Lịch sử đăng nhập web có phải ghi DB không? | Compliance | S12 |
| Q6 | Refresh token web sống bao lâu (đề xuất 1 ngày)? | Product | S0 |
| Q7 | Luồng **Quên mật khẩu** trên web đi đường nào? | Product + Architect | ngoài phạm vi task này |
