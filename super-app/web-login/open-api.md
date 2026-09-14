# API Spec — Luồng Đăng nhập Web (OpenAPI 3.0.3)

| | |
|---|---|
| **Service** | `authentication-service` |
| **Context path** | `/api/authentication` |
| **Phiên bản spec** | 1.0 — 2026-09-14 |
| **Trạng thái** | Draft — hợp đồng chính thức giữa BE và FE |
| **Tài liệu liên quan** | [`web-login-design.md`](web-login-design.md) (thiết kế) · [`web-login-task.md`](web-login-task.md) (các bước triển khai) |

---

## 1. Quy ước chung

### 1.1 Envelope

Mọi response bọc trong `ResponseApi` của `common-libs` (`com.finx.spring.service.api.ResponseApi`) — **3 trường cấp cao**:

```json
{
  "status": { "code": "SUCCESS", "message": null, "errors": [] },
  "payload": { },
  "meta":   { "requestId": "b7c1...", "nextCursor": null }
}
```

| Trường | Ý nghĩa |
|---|---|
| `status.code` | `SUCCESS` khi thành công; khi lỗi là **tên mã lỗi** dạng chuỗi (ví dụ `CAPTCHA_INVALID`) |
| `status.message` | Mô tả lỗi (tiếng Anh, dành cho log — **không** hiển thị thẳng cho người dùng) |
| `status.errors[]` | Danh sách `{ field, message }` khi lỗi validate từng trường |
| `payload` | Dữ liệu nghiệp vụ; `null` khi lỗi |
| `meta.requestId` | Lấy từ MDC `X-Request-ID` — **dùng giá trị này khi báo lỗi cho BE** |

> ⚠️ FE **phải** đọc `status.code`, không dựa vào HTTP status để phân biệt loại lỗi nghiệp vụ.

### 1.2 Header

| Header | Bắt buộc | Ghi chú |
|---|---|---|
| `Content-Type: application/json` | ✅ (POST) | |
| `GTW-Authorization` | ✅ | Token pre-login. **Cơ chế cấp cho web đang chờ Infra chốt** — xem `web-login-task.md` Q2 |
| `X-Request-ID` | khuyến nghị | Truy vết xuyên service; nếu không gửi, BE tự sinh |
| `X-Forwarded-For` | tự động | Do ALB/CDN thêm; là khoá rate limit cho `/web-login` |

> Luồng web **không** dùng header `device-id`, `X-Platform`, `app-version` như mobile, và **không** ký payload ED25519.

### 1.3 Thứ tự gọi

```
GET  /v1/auth/captcha                     →  captchaId + ảnh
POST /v1/auth/web-login                   →  sessionId + thông tin OTP
POST /v1/auth/web-login/resend-otp        →  (tuỳ chọn, khi bấm "Gửi lại mã")
POST /v1/auth/web-login/verify-otp        →  accessToken + refreshToken
```

---

## 2. OpenAPI 3.0.3
```yaml
openapi: 3.0.3
info:
  title: Vikki Web Login API
  description: |
    Luồng đăng nhập dành riêng cho Vikki Web: CAPTCHA (tự implement, lưu Redis)
    → số điện thoại + mật khẩu → SMS OTP → cấp JWT (hết hạn 30 phút).
    Token do AWS Cognito phát hành qua app client riêng cho web, dùng chung user pool với mobile.
  version: 1.0.0
  contact:
    name: Vikki Platform - authentication-service

servers:
  - url: https://{host}/api/authentication
    description: API Gateway phục vụ Vikki Web
    variables:
      host:
        default: api-ext.prod.galaxyfinx.in

tags:
  - name: Web Login
    description: 4 endpoint của luồng đăng nhập web

security:
  - GtwAuthorization: []

paths:

  /v1/auth/captcha:
    get:
      tags: [Web Login]
      operationId: issueWebLoginCaptcha
      summary: Phát hành ảnh CAPTCHA (kiêm re-render)
      description: |
        Trả về `captchaId` và ảnh PNG dạng data URI để gắn thẳng vào `<img src>`.
        Bấm nút ↻ trên UI thì gọi lại chính endpoint này kèm `previousCaptchaId`
        để huỷ mã cũ ngay lập tức.

        Giới hạn: 20 mã/phút/IP.
      parameters:
        - name: previousCaptchaId
          in: query
          required: false
          description: Mã CAPTCHA trước đó cần huỷ (khi người dùng bấm re-render)
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Phát hành thành công
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/ResponseApi'
                  - type: object
                    properties:
                      payload:
                        $ref: '#/components/schemas/CaptchaResponse'
              examples:
                success:
                  value:
                    status: { code: SUCCESS, message: null, errors: [] }
                    payload:
                      captchaId: 8b2f4b7c-9d31-4f0a-bb2e-6c1f0a91c3d7
                      image: "data:image/png;base64,iVBORw0KGgoAAAANSUhEUg..."
                      expiresIn: 120
                    meta: { requestId: 6f0c1d2e-..., nextCursor: null }
        '429':
          description: Vượt giới hạn phát hành CAPTCHA
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ErrorResponse' }
              examples:
                rateLimited:
                  value:
                    status: { code: CAPTCHA_RATE_LIMITED, message: "Too many captcha requests", errors: [] }
                    payload: null
                    meta: { requestId: 6f0c1d2e-..., nextCursor: null }

  /v1/auth/web-login:
    post:
      tags: [Web Login]
      operationId: webLoginStepOne
      summary: Bước 1 — số điện thoại + mật khẩu + CAPTCHA
      description: |
        Xác minh CAPTCHA → tra CIF theo số điện thoại → kiểm tra trạng thái khách hàng →
        xác minh mật khẩu qua Cognito (web app client) → tạo phiên đăng nhập web →
        yêu cầu mfa-service gửi SMS OTP.

        **Không** trả về token ở bước này. Token chỉ được cấp ở `/verify-otp`.

        CAPTCHA là **dùng một lần**: sai hay đúng đều bị huỷ. Khi nhận `CAPTCHA_INVALID`,
        FE phải tự động gọi lại `GET /v1/auth/captcha`.

        Rate limit: 10 request/5 phút/IP, vượt thì chặn 15 phút.
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/WebLoginRequest' }
            examples:
              default:
                value:
                  countryCode: "+84"
                  phoneNumber: "908822911"
                  password: "Vmlra2k5OTk="
                  captchaId: 8b2f4b7c-9d31-4f0a-bb2e-6c1f0a91c3d7
                  captchaAnswer: a3er7p
      responses:
        '200':
          description: Mật khẩu đúng, OTP đã được gửi
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/ResponseApi'
                  - type: object
                    properties:
                      payload:
                        $ref: '#/components/schemas/WebLoginResponse'
              examples:
                success:
                  value:
                    status: { code: SUCCESS, message: null, errors: [] }
                    payload:
                      sessionId: 3f8a5c22-7b41-4e0d-9a11-2d7c5e8f01ab
                      maskedPhoneNumber: "+84908822911"
                      otpExpiresIn: 90
                      resendAvailableIn: 30
                      remainingResend: 4
                    meta: { requestId: 6f0c1d2e-..., nextCursor: null }
        '400':
          description: CAPTCHA sai/hết hạn, không tìm thấy số điện thoại, sai mật khẩu, mật khẩu hết hạn
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ErrorResponse' }
              examples:
                captchaInvalid:
                  value:
                    status: { code: CAPTCHA_INVALID, message: "Captcha is incorrect", errors: [] }
                    payload: null
                    meta: { requestId: 6f0c1d2e-..., nextCursor: null }
                wrongPassword:
                  value:
                    status: { code: FAILED_AUTHENTICATION, message: "Authentication failed", errors: [] }
                    payload: { remainingAttempts: 3 }
                    meta: { requestId: 6f0c1d2e-..., nextCursor: null }
        '429':
          description: Vượt rate limit theo IP, hoặc mfa-service chặn sinh OTP
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ErrorResponse' }

  /v1/auth/web-login/resend-otp:
    post:
      tags: [Web Login]
      operationId: resendWebLoginOtp
      summary: Gửi lại mã OTP
      description: |
        Tương ứng link **"Gửi lại mã"** trên màn hình nhập OTP.

        Hai lớp giới hạn: cooldown **30 giây** giữa 2 lần, và tối đa **5 lần/giờ**
        do mfa-service áp (variant `OTP_AUTH`).

        Gửi lại **không** gia hạn thời gian sống của phiên đăng nhập.
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/ResendOtpRequest' }
            examples:
              default:
                value: { sessionId: 3f8a5c22-7b41-4e0d-9a11-2d7c5e8f01ab }
      responses:
        '200':
          description: Đã gửi lại OTP
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/ResponseApi'
                  - type: object
                    properties:
                      payload:
                        $ref: '#/components/schemas/ResendOtpResponse'
              examples:
                success:
                  value:
                    status: { code: SUCCESS, message: null, errors: [] }
                    payload: { otpExpiresIn: 90, resendAvailableIn: 30, remainingResend: 3 }
                    meta: { requestId: 6f0c1d2e-..., nextCursor: null }
        '400':
          description: Phiên đăng nhập đã hết hạn hoặc đã dùng
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ErrorResponse' }
              examples:
                expired:
                  value:
                    status: { code: SESSION_EXPIRED, message: "Web login session expired", errors: [] }
                    payload: null
                    meta: { requestId: 6f0c1d2e-..., nextCursor: null }
        '429':
          description: Còn cooldown, hoặc vượt số lần gửi lại cho phép
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ErrorResponse' }
              examples:
                tooSoon:
                  value:
                    status: { code: OTP_RESEND_TOO_SOON, message: "Please wait before requesting a new OTP", errors: [] }
                    payload: { resendAvailableIn: 17 }
                    meta: { requestId: 6f0c1d2e-..., nextCursor: null }

  /v1/auth/web-login/verify-otp:
    post:
      tags: [Web Login]
      operationId: verifyWebLoginOtp
      summary: Bước 2 — xác minh OTP và cấp JWT
      description: |
        Xác minh OTP qua mfa-service. Đúng thì trả về token đã giữ sẵn trong phiên
        và **huỷ phiên ngay** — một `sessionId` chỉ đổi được **một** bộ token.

        OTP sai **không** huỷ phiên: người dùng nhập lại hoặc bấm gửi lại mã.

        ⚠️ **`expiresIn` là thời hạn CÒN LẠI, không phải 1800.** Token được Cognito
        phát hành từ bước 1 nên đồng hồ 30 phút đã chạy trong lúc người dùng nhập OTP.
        FE **phải** hẹn lịch gọi `refresh-token` theo giá trị này.
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/VerifyOtpRequest' }
            examples:
              default:
                value:
                  sessionId: 3f8a5c22-7b41-4e0d-9a11-2d7c5e8f01ab
                  otp: "222222"
      responses:
        '200':
          description: Đăng nhập thành công
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/ResponseApi'
                  - type: object
                    properties:
                      payload:
                        $ref: '#/components/schemas/WebLoginTokenResponse'
              examples:
                success:
                  value:
                    status: { code: SUCCESS, message: null, errors: [] }
                    payload:
                      accessToken: "eyJraWQiOiJ..."
                      refreshToken: "eyJjdHkiOiJ..."
                      expiresIn: 1735
                      isPinExpired: false
                      isFirstLogin: false
                    meta: { requestId: 6f0c1d2e-..., nextCursor: null }
        '400':
          description: OTP sai, hoặc phiên hết hạn / đã dùng
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ErrorResponse' }
              examples:
                wrongOtp:
                  value:
                    status: { code: INVALID_CHALLENGE_OTP, message: "Invalid otp", errors: [] }
                    payload: null
                    meta: { requestId: 6f0c1d2e-..., nextCursor: null }
                sessionExpired:
                  value:
                    status: { code: SESSION_EXPIRED, message: "Web login session expired", errors: [] }
                    payload: null
                    meta: { requestId: 6f0c1d2e-..., nextCursor: null }
        '429':
          description: Nhập sai OTP quá số lần cho phép, tài khoản bị khoá OTP 1 giờ
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ErrorResponse' }

components:

  securitySchemes:
    GtwAuthorization:
      type: apiKey
      in: header
      name: GTW-Authorization
      description: Token pre-login do API Gateway/authorizer cấp trước khi đăng nhập.

  schemas:

    ResponseStatus:
      type: object
      required: [code]
      properties:
        code:
          type: string
          description: '`SUCCESS` khi thành công, ngược lại là tên mã lỗi'
          example: SUCCESS
        message:
          type: string
          nullable: true
        errors:
          type: array
          items: { $ref: '#/components/schemas/FieldError' }

    FieldError:
      type: object
      properties:
        field: { type: string, example: phoneNumber }
        message: { type: string, example: Format phone number is not correct }

    ResponseMeta:
      type: object
      properties:
        requestId: { type: string, nullable: true }
        nextCursor: { type: string, nullable: true }

    ResponseApi:
      type: object
      properties:
        status: { $ref: '#/components/schemas/ResponseStatus' }
        payload: { type: object, nullable: true }
        meta: { $ref: '#/components/schemas/ResponseMeta' }

    ErrorResponse:
      type: object
      properties:
        status: { $ref: '#/components/schemas/ResponseStatus' }
        payload:
          type: object
          nullable: true
          description: Một số lỗi kèm dữ liệu phụ, ví dụ `remainingAttempts`, `resendAvailableIn`
        meta: { $ref: '#/components/schemas/ResponseMeta' }

    CaptchaResponse:
      type: object
      required: [captchaId, image, expiresIn]
      properties:
        captchaId:
          type: string
          format: uuid
          description: Định danh mã CAPTCHA, gửi kèm ở bước `/web-login`
        image:
          type: string
          description: Ảnh PNG dạng data URI, gắn thẳng vào thuộc tính `src` của thẻ `img`
          example: "data:image/png;base64,iVBORw0KGgo..."
        expiresIn:
          type: integer
          format: int32
          description: Số giây còn hiệu lực
          example: 120

    WebLoginRequest:
      type: object
      required: [countryCode, phoneNumber, password, captchaId, captchaAnswer]
      properties:
        countryCode:
          type: string
          description: Mã quốc gia, kèm dấu `+`
          example: "+84"
        phoneNumber:
          type: string
          description: Số điện thoại, không gồm mã quốc gia
          example: "908822911"
        password:
          type: string
          format: byte
          description: |
            Mật khẩu mã hoá **Base64** (thống nhất với `/v3/auth/login` của mobile).
            Ví dụ `Vikki999` → `Vmlra2k5OTk=`. Base64 sai định dạng ⇒ `INVALID_INPUT`.
          example: Vmlra2k5OTk=
        captchaId:
          type: string
          format: uuid
        captchaAnswer:
          type: string
          description: Người dùng nhập; so khớp **không phân biệt hoa/thường**
          minLength: 6
          maxLength: 6
          example: a3er7p

    WebLoginResponse:
      type: object
      required: [sessionId, maskedPhoneNumber, otpExpiresIn]
      properties:
        sessionId:
          type: string
          format: uuid
          description: Định danh phiên đăng nhập web; dùng cho `/resend-otp` và `/verify-otp`
        maskedPhoneNumber:
          type: string
          description: Số điện thoại hiển thị trên màn hình nhập OTP
          example: "+84908822911"
        otpExpiresIn:
          type: integer
          description: Số giây OTP còn hiệu lực — FE dùng để đếm ngược
          example: 90
        resendAvailableIn:
          type: integer
          description: Số giây phải chờ trước khi được bấm "Gửi lại mã"
          example: 30
        remainingResend:
          type: integer
          description: Số lần gửi lại còn được phép trong giờ hiện tại
          example: 4

    ResendOtpRequest:
      type: object
      required: [sessionId]
      properties:
        sessionId: { type: string, format: uuid }

    ResendOtpResponse:
      type: object
      properties:
        otpExpiresIn: { type: integer, example: 90 }
        resendAvailableIn: { type: integer, example: 30 }
        remainingResend: { type: integer, example: 3 }

    VerifyOtpRequest:
      type: object
      required: [sessionId, otp]
      properties:
        sessionId: { type: string, format: uuid }
        otp:
          type: string
          pattern: '^[0-9]{6}$'
          description: Mã OTP 6 chữ số nhận qua SMS
          example: "222222"

    WebLoginTokenResponse:
      type: object
      required: [accessToken, refreshToken, expiresIn]
      properties:
        accessToken:
          type: string
          description: JWT Cognito, dùng cho các API nghiệp vụ
        refreshToken:
          type: string
        expiresIn:
          type: integer
          description: |
            **Thời hạn còn lại** của access token, tính bằng giây — luôn **nhỏ hơn** 1800
            vì token được phát hành từ bước `/web-login`.
          example: 1735
        isPinExpired:
          type: boolean
          nullable: true
        isFirstLogin:
          type: boolean
          nullable: true
```

---

## 3. Bảng mã lỗi

### 3.1 Mã lỗi mới của luồng web

| `status.code` | HTTP | Thông điệp gợi ý hiển thị | Hành động FE |
|---|---|---|---|
| `CAPTCHA_INVALID` | 400 | "Mã xác nhận không đúng" | Tự động gọi lại `GET /v1/auth/captcha` |
| `CAPTCHA_EXPIRED` | 400 | "Mã xác nhận đã hết hạn" | Tự động lấy mã mới |
| `CAPTCHA_RATE_LIMITED` | 429 | "Vui lòng thử lại sau" | Khoá nút refresh 60 giây |
| `SESSION_EXPIRED` | 400 | "Phiên đăng nhập đã hết hạn, vui lòng đăng nhập lại" | Quay về màn nhập mật khẩu, lấy captcha mới |
| `OTP_RESEND_TOO_SOON` | 429 | "Vui lòng đợi {n} giây để gửi lại" | Đọc `payload.resendAvailableIn` để đếm ngược |
| `OTP_GENERATION_LIMIT_EXCEEDED` | 429 | "Bạn đã yêu cầu quá số lần cho phép" | Ẩn nút gửi lại |
| `OTP_VERIFICATION_LIMIT_EXCEEDED` | 429 | "Nhập sai quá số lần, vui lòng thử lại sau 1 giờ" | Quay về màn đăng nhập |

### 3.2 Mã lỗi tái sử dụng (đã có trong `ErrorCode.java`)

| `status.code` | HTTP | Bối cảnh | Thông điệp gợi ý |
|---|---|---|---|
| `PHONE_NUMBER_NOT_FOUND` | 400 | Số điện thoại không có trong hệ thống | "Thông tin đăng nhập không đúng" |
| `FAILED_AUTHENTICATION` | 400 | Sai mật khẩu (kèm `payload.remainingAttempts`) | "Thông tin đăng nhập không đúng" |
| `NOT_ALLOW_LOGIN_IN_LOCKED_TIME` | 400 | Tài khoản đang bị khoá do sai mật khẩu nhiều lần | "Tài khoản tạm khoá, vui lòng thử lại sau" |
| `USER_PASSWORD_EXPIRED` | 400 | Mật khẩu hết hạn | Điều hướng sang đổi mật khẩu |
| `CUSTOMER_RESTRICTED_COUNTER_ONLY` | 400 | Khách hàng bị hạn chế, chỉ giao dịch tại quầy | "Vui lòng liên hệ chi nhánh" |
| `INVALID_CHALLENGE_OTP` | 400 | OTP sai | "Mã OTP không khớp, vui lòng thử lại" |
| `INVALID_INPUT` | 400 | Mật khẩu không phải Base64 hợp lệ | "Dữ liệu không hợp lệ" |
| `VALIDATION_FAILED` | 400 | Sai định dạng field (xem `status.errors[]`) | Hiển thị lỗi theo từng ô |
| `SERVER_ERROR` | 500 | Lỗi hệ thống | "Có lỗi xảy ra, vui lòng thử lại" |

> 🔒 **Lưu ý chống dò tài khoản:** `PHONE_NUMBER_NOT_FOUND` và `FAILED_AUTHENTICATION` nên hiển thị **cùng một** thông điệp trên UI, dù mã lỗi khác nhau. (Việc có gộp mã ở BE hay không đang chờ Security chốt — `web-login-task.md` Q3.)

---

## 4. Ví dụ gọi bằng `curl`

```bash
BASE="https://api-ext.prod.galaxyfinx.in/api/authentication"
GTW="<pre-login-token>"

# 1) Lấy CAPTCHA
curl -s "$BASE/v1/auth/captcha" -H "GTW-Authorization: $GTW"

# 1b) Bấm ↻ - huỷ mã cũ, lấy mã mới
curl -s "$BASE/v1/auth/captcha?previousCaptchaId=8b2f4b7c-..." -H "GTW-Authorization: $GTW"

# 2) Bước 1 - phone + password + captcha  (password = base64 của "Vikki999")
curl -s -X POST "$BASE/v1/auth/web-login" \
  -H "GTW-Authorization: $GTW" -H "Content-Type: application/json" \
  -d '{
        "countryCode": "+84",
        "phoneNumber": "908822911",
        "password": "Vmlra2k5OTk=",
        "captchaId": "8b2f4b7c-9d31-4f0a-bb2e-6c1f0a91c3d7",
        "captchaAnswer": "a3er7p"
      }'

# 3) Gửi lại mã (tuỳ chọn)
curl -s -X POST "$BASE/v1/auth/web-login/resend-otp" \
  -H "GTW-Authorization: $GTW" -H "Content-Type: application/json" \
  -d '{"sessionId":"3f8a5c22-7b41-4e0d-9a11-2d7c5e8f01ab"}'

# 4) Bước 2 - xác minh OTP, nhận token
curl -s -X POST "$BASE/v1/auth/web-login/verify-otp" \
  -H "GTW-Authorization: $GTW" -H "Content-Type: application/json" \
  -d '{"sessionId":"3f8a5c22-7b41-4e0d-9a11-2d7c5e8f01ab","otp":"222222"}'
```

---

## 5. Ghi chú tích hợp cho Front-end

| # | Yêu cầu | Chi tiết |
|---|---|---|
| FE1 | Mật khẩu phải **Base64-encode** trước khi gửi | `btoa("Vikki999")` → `"Vmlra2k5OTk="` |
| FE2 | Ảnh CAPTCHA gắn thẳng `<img src={payload.image}>` | Đã là data URI, không cần xử lý thêm |
| FE3 | Nút ↻ gọi lại `GET /captcha` kèm `previousCaptchaId` | Mã cũ mất hiệu lực ngay |
| FE4 | Nhận `CAPTCHA_INVALID`/`CAPTCHA_EXPIRED` ⇒ **tự động** lấy mã mới | CAPTCHA dùng một lần, không cho nhập lại cùng mã |
| FE5 | Đếm ngược OTP theo `otpExpiresIn` (90 giây) | Khớp dòng chữ trên mockup |
| FE6 | Nút "Gửi lại mã" chỉ bật sau `resendAvailableIn` giây | Ẩn hẳn khi `remainingResend = 0` |
| FE7 | OTP sai ⇒ **giữ nguyên** `sessionId`, cho nhập lại | Không gọi lại `/web-login` |
| FE8 | Hẹn lịch `refresh-token` theo `expiresIn` **trả về**, không hardcode 1800 | Token đã tiêu hao thời gian ở bước nhập OTP |
| FE9 | Lưu `accessToken` trong **bộ nhớ**, refresh token trong cookie `HttpOnly; Secure; SameSite=Strict` | Khuyến nghị bảo mật, chờ FE xác nhận |
| FE10 | Khi báo lỗi cho BE, gửi kèm `meta.requestId` | Truy vết log xuyên service |

---

## 6. Vòng đời phiên sau khi đăng nhập

| Việc | API | Ghi chú |
|---|---|---|
| Gia hạn token | `POST /api/authentication/v1/auth/refresh-token` | **Tái sử dụng** API sẵn có |
| Đăng xuất | `POST /api/authentication/v1/auth/sign-out` | **Tái sử dụng** API sẵn có |
| Quên mật khẩu | `POST /api/authentication/v5/auth/forgot-password` | Link "Quên mật khẩu?" trên mockup — **cần khảo sát riêng cho web** (chain hiện tại có bước FACE/NFC cần SDK) |
