# CFM 4 / NiFi 2.x Clipboard Copy & Paste 문제 정리

## 1. 개요

Cloudera CFM 4 환경에서 NiFi UI의 **Copy / Paste 기능이 동작하지 않는 현상**이 발생할 수 있다.

해당 현상은 NiFi의 권한 설정이나 Non-Secure Cluster 자체의 문제가 아니라, **NiFi 2.x에서 사용하는 Browser Clipboard API와 브라우저의 보안 정책에 의해 발생하는 현상**이다.

특히 NiFi를 다음과 같이 **HTTP 기반 Non-Secure 환경**으로 사용하는 경우 발생할 수 있다.

```text
http://node3.cfm4-jjin.coelab.cloudera.com:8080
```

---

## 2. 원인

### 2.1 NiFi 권한 문제는 아님

Non-Secure NiFi Cluster에서는 별도의 Authentication 및 Authorization을 수행하지 않기 때문에 NiFi UI상의 작업이 사용자 권한에 의해 제한되는 구조가 아니다.

따라서 Copy / Paste가 동작하지 않는 현상은 NiFi의 다음 설정 문제로 보기 어렵다.

- User Authentication
- User Authorization
- Access Policy
- Processor 또는 Process Group 권한

즉, **NiFi 자체의 권한 설정 문제가 아니라 Browser 측 보안 정책이 원인**이다.

---

## 3. NiFi 2.x의 Clipboard 동작 방식

CFM 4의 NiFi 2.x에서는 Copy / Paste 기능을 수행하기 위해 Browser의 Clipboard API를 사용한다.

대표적인 API는 다음과 같다.

```javascript
navigator.clipboard
```

NiFi UI에서 Copy 동작을 수행하면 내부적으로 Browser의 Clipboard API를 통해 데이터를 Clipboard에 저장한다.

```text
NiFi UI
   │
   │ Copy
   ▼
navigator.clipboard
   │
   ▼
Browser Clipboard
```

---

## 4. Browser Secure Context 제한

Chrome을 비롯한 최신 Browser에서는 `navigator.clipboard` API를 보안상의 이유로 **Secure Context에서만 사용할 수 있도록 제한**하고 있다.

일반적으로 HTTPS로 접속한 웹 사이트가 Secure Context로 간주된다.

예를 들어 다음 URL은 Secure Context이다.

```text
https://nifi.example.com
```

반면 다음과 같은 일반 HTTP URL은 Secure Context가 아니다.

```text
http://nifi.example.com:8080
```

따라서 HTTP 기반 NiFi에 접속한 경우 Browser가 Clipboard API 사용을 제한할 수 있다.

동작 구조는 다음과 같다.

```text
CFM 4 / NiFi 2.x
        │
        │ Copy / Paste
        ▼
navigator.clipboard
        │
        ▼
Browser Secure Context 확인
        │
        ├── HTTPS
        │     │
        │     ▼
        │ Clipboard API 허용
        │     │
        │     ▼
        │ Copy / Paste 정상
        │
        └── HTTP
              │
              ▼
        Clipboard API 제한
              │
              ▼
        Copy / Paste 실패
```

---

## 5. 접속 방식별 동작

| NiFi 접속 방식 | Secure Context | Clipboard API | Copy / Paste |
|---|---|---|---|
| `https://nifi.example.com` | Yes | 사용 가능 | 정상 |
| `http://localhost:8080` | Browser 예외 처리 가능 | 사용 가능할 수 있음 | 정상 가능 |
| `http://hostname:8080` | No | 제한 | 실패 가능 |
| `http://IP:8080` | No | 제한 | 실패 가능 |

따라서 CFM 4의 NiFi를 HTTP 기반으로 접속하는 경우 Copy / Paste가 동작하지 않을 수 있다.

---

# 6. 해결 방법

해결 방법은 크게 두 가지가 있다.

1. Chrome에서 해당 HTTP NiFi URL을 Secure Origin으로 강제 처리
2. NiFi에 TLS를 적용하여 HTTPS로 접속

---

# 7. 방법 1 - Chrome 설정 변경

테스트 또는 개발 환경에서는 Chrome 설정을 변경하여 특정 HTTP URL을 Secure Context로 취급하도록 할 수 있다.

Cloudera 측에서도 테스트 환경에서 해당 설정을 통해 Copy / Paste가 정상적으로 동작하는 것을 확인하였다.

## 7.1 Chrome Flags 페이지 접속

Chrome 주소창에 다음 URL을 입력한다.

```text
chrome://flags/#unsafely-treat-insecure-origin-as-secure
```

---

## 7.2 Insecure origins treated as secure 활성화

다음 항목을 찾는다.

```text
Insecure origins treated as secure
```

설정 값을 다음과 같이 변경한다.

```text
Enabled
```

---

## 7.3 NiFi URL 등록

설정 항목의 Text Box에 NiFi URL을 입력한다.

예:

```text
http://node3.cfm4-jjin.coelab.cloudera.com:8080
```

Browser에서는 URL이 아니라 **Origin 단위**로 처리한다.

Origin은 다음 세 가지 요소의 조합이다.

```text
Protocol + Host + Port
```

예:

```text
http://node3.cfm4-jjin.coelab.cloudera.com:8080
```

각 요소는 다음과 같다.

```text
Protocol : http
Host     : node3.cfm4-jjin.coelab.cloudera.com
Port     : 8080
```

---

## 7.4 Cluster의 여러 Node에 직접 접속하는 경우

NiFi Cluster의 각 Node URL에 직접 접속해야 한다면 각각의 Origin을 등록해야 할 수 있다.

예:

```text
http://node1.cfm4-jjin.coelab.cloudera.com:8080
http://node2.cfm4-jjin.coelab.cloudera.com:8080
http://node3.cfm4-jjin.coelab.cloudera.com:8080
```

각 URL은 Browser 관점에서 서로 다른 Origin이다.

---

## 7.5 Chrome 재시작

설정 완료 후 Chrome 화면 하단의 다음 버튼을 클릭한다.

```text
Relaunch
```

Chrome이 재시작되면 NiFi에 다시 접속하여 Copy / Paste를 테스트한다.

---

## 7.6 설정 절차 요약

```text
1. Chrome 실행

2. 아래 URL 접속

   chrome://flags/#unsafely-treat-insecure-origin-as-secure

3. "Insecure origins treated as secure" 설정

   Disabled
      ↓
   Enabled

4. NiFi URL 등록

   http://node3.cfm4-jjin.coelab.cloudera.com:8080

5. Chrome Relaunch

6. NiFi 재접속

7. Copy / Paste 테스트
```

---

# 8. 방법 2 - NiFi TLS 구성

Browser 설정을 변경하지 않으려면 NiFi에 TLS를 구성하여 HTTPS로 접속하도록 해야 한다.

기존 접속 방식이 다음과 같다면:

```text
http://nifi.example.com:8080
```

TLS 구성 후 다음과 같이 변경한다.

```text
https://nifi.example.com
```

또는 구성에 따라:

```text
https://nifi.example.com:8443
```

HTTPS를 사용하면 Browser에서 해당 NiFi 페이지를 Secure Context로 인식한다.

따라서 다음 API를 정상적으로 사용할 수 있다.

```javascript
navigator.clipboard
```

결과적으로 Copy / Paste 기능도 정상적으로 동작하게 된다.

---

# 9. TLS와 Authentication / Authorization

Cloudera 답변에서는 Browser 설정을 변경하지 않는 경우 다음과 같은 Secure NiFi 구성을 권고하고 있다.

```text
TLS
+
Authentication
+
Authorization
```

즉 단순히 Clipboard 문제를 해결하는 것뿐만 아니라 NiFi 자체를 Secure Mode로 구성하는 방식이다.

구조적으로는 다음과 같다.

```text
User
 │
 │ HTTPS
 ▼
NiFi
 │
 ├── TLS
 │
 ├── Authentication
 │
 └── Authorization
```

이를 통해 다음을 함께 확보할 수 있다.

- HTTPS 통신
- Browser Secure Context
- Clipboard API 정상 사용
- 사용자 인증
- 사용자별 권한 관리
- NiFi UI 보안 강화

---

# 10. 해결 방법 비교

| 구분 | Chrome 설정 변경 | NiFi TLS 구성 |
|---|---|---|
| 적용 위치 | 사용자 Browser | NiFi Server |
| 적용 난이도 | 낮음 | 상대적으로 높음 |
| Browser별 설정 필요 | Yes | No |
| HTTPS 필요 | No | Yes |
| Authentication 구성 | 불필요 | 일반적으로 구성 |
| Authorization 구성 | 불필요 | 일반적으로 구성 |
| 테스트 환경 | 적합 | 가능 |
| 운영 환경 | 비권장 | 권장 |
| 근본적인 해결 | 임시 우회 | Yes |

---

# 11. 환경별 권장 방법

| 환경 | 권장 방법 |
|---|---|
| 개인 개발 환경 | Chrome Flag 설정 |
| PoC 환경 | Chrome Flag 또는 TLS |
| 내부 테스트 환경 | Chrome Flag 사용 가능 |
| 개발/검증 환경 | 가능하면 TLS |
| Production 환경 | TLS + Authentication + Authorization |

특히 다음 Chrome 설정은:

```text
unsafely-treat-insecure-origin-as-secure
```

이름 그대로 **보안되지 않은 HTTP Origin을 Browser에서 강제로 Secure Context로 처리하는 설정**이다.

따라서 운영환경에서 여러 사용자의 Browser에 해당 설정을 적용하는 방식보다는 NiFi 자체에 TLS를 구성하는 것이 적절하다.

---

# 12. 전체 문제 구조

```text
                     CFM 4
                    NiFi 2.x
                       │
                       │
                 Copy / Paste
                       │
                       ▼
              navigator.clipboard
                       │
                       ▼
            Browser Secure Context
                       │
          ┌────────────┴────────────┐
          │                         │
        HTTPS                      HTTP
          │                         │
          ▼                         ▼
   Secure Context            Insecure Context
          │                         │
          ▼                         ▼
 Clipboard API 허용          Clipboard API 제한
          │                         │
          ▼                         ▼
 Copy / Paste 정상          Copy / Paste 실패
                                    │
                    ┌───────────────┴──────────────┐
                    │                              │
                    ▼                              ▼
          Chrome Flag 설정                  NiFi TLS 구성
                    │                              │
                    ▼                              ▼
          HTTP를 Secure로 간주                 HTTPS 사용
                    │                              │
                    └───────────────┬──────────────┘
                                    ▼
                           Copy / Paste 정상
```

---

# 13. 결론

CFM 4 / NiFi 2.x에서 발생하는 Copy / Paste 문제는 **Non-Secure NiFi의 사용자 권한 문제라기보다는 Browser Clipboard API의 Secure Context 제한으로 인해 발생하는 문제**이다.

NiFi 2.x에서는 Copy / Paste를 처리하기 위해 다음 Browser API를 사용한다.

```javascript
navigator.clipboard
```

Chrome 등의 Browser에서는 해당 API를 기본적으로 HTTPS와 같은 Secure Context에서 사용하도록 제한하고 있기 때문에 HTTP 기반 Non-Secure NiFi 환경에서는 Clipboard 기능이 차단될 수 있다.

단기적으로는 Chrome의 다음 설정을 이용하여 해결할 수 있다.

```text
chrome://flags/#unsafely-treat-insecure-origin-as-secure
```

해당 설정에서 NiFi HTTP URL을 Secure Origin으로 등록하면 Copy / Paste를 사용할 수 있다.

하지만 이는 Browser 측의 예외 처리이므로 **운영 환경의 근본적인 해결 방법은 NiFi TLS를 구성하고 HTTPS를 사용하는 것**이다.

### 권장 방향

```text
개발 / 테스트
    │
    └── Chrome Flag 설정 사용 가능

운영 환경
    │
    └── TLS + Authentication + Authorization
          │
          └── HTTPS 기반 NiFi 접속
```
