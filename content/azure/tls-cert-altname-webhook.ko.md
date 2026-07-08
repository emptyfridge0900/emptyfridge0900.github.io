+++
title = "웹훅이 핸드쉐이크에서 죽었다: ERR_TLS_CERT_ALTNAME_INVALID 디버깅"
date = 2026-06-22

[taxonomies]
categories = ["post"]
tags = ["tls", "ssl", "certificate", "webhook", "dns", "debugging"]
+++

Node.js 서비스가 웹훅을 백엔드 API로 전달하다가 이런 에러를 뱉었다.

```
[onRaw] webhook forwarding error: {
  name: 'ApiError',
  title: 'Webhook forwarding failed',
  status: 502,
  message: "Hostname/IP does not match certificate's altnames:
            Host: example.com. is not in the cert's altnames:
            DNS:*.azurewebsites.net, DNS:*.scm.azurewebsites.net, ...",
  code: 'ERR_TLS_CERT_ALTNAME_INVALID'
}
```

페이로드 자체는 멀쩡했다. 데이터 문제가 아니라는 거다. 근데 왜 502가 뜨는 걸까?

## 원인: SAN에 없는 도메인

포워더가 `example.com`으로 HTTPS 연결을 열었다. TLS 핸드셰이크 과정에서 서버가 돌려준 인증서의 **SAN(Subject Alternative Names)** 목록에 `example.com`이 없었다. 위 에러에서는 `*.azurewebsites.net` 계열 이름들만 들어있는 걸 볼 수 있는데 — 이건 Azure 특유의 이야기가 아니다. 어떤 플랫폼이든, 어떤 서버든, 커스텀 도메인을 *연결*만 해놓고 그 도메인을 커버하는 인증서를 *바인딩*하지 않으면 똑같은 에러가 난다.

TLS 클라이언트는 "내가 연결하려는 호스트명이 인증서에 있나?"를 필수로 체크하는데, 없으니까 `ERR_TLS_CERT_ALTNAME_INVALID`로 연결을 끊어버린다. 포워더는 이걸 502로 감싸서 올려보낸 것이다.

**한 문장 요약:**
> `example.com`으로 트래픽이 *라우팅*되도록 설정은 했지만, `example.com`을 커버하는 TLS 인증서는 *바인딩하지 않아서* — 서버가 기본 인증서(플랫폼/서버의 다른 이름)로 폴백했고, 클라이언트가 연결한 이름과 맞지 않았다.

## 핵심 개념: DNS(라우팅) ≠ TLS(신원증명)

이 두 개를 헷갈리면 계속 막힌다.

| 레이어 | 묻는 질문 | 동작 방식 |
|--------|----------|---------|
| **DNS / 라우팅** | "트래픽을 *어디로* 보내지?" | A / CNAME 레코드 |
| **TLS / 신원** | "서버가 `example.com`임을 *증명*할 수 있나?" | `example.com`이 포함된 인증서 |

> **도메인으로 트래픽이 들어온다고 그 도메인의 인증서가 생기는 게 아니다.** 인증서는 따로 발급받고, 따로 바인딩해야 한다.

비유하자면, DNS는 회사 전화번호부다 — 어디로 연결하면 되는지 알려준다. TLS 인증서는 직원증이다. `example.com`으로 편지가 도착했다고 해서 그 사람이 `example.com`이라고 적힌 신분증을 들고 있는 건 아니다. 경비(TLS 검사)가 신분증을 확인하고, 없으면 돌려보낸다.

그리고 핸드셰이크가 **HTTP 요청 전에** 실패하기 때문에 백엔드 애플리케이션에는 아무 로그도 안 찍힌다. 서비스 코드 디버깅은 의미가 없다. 순수하게 인프라/인증서 문제다.

## 수정 방법: 인증서 발급 + 바인딩

서버가 어디에 있든 수정 방향은 동일하다.

### 인증서 vs 바인딩

이 둘은 별개다.

| 객체 | 정체 |
|------|------|
| **인증서** | "`example.com`임을 증명하는" 서명된 파일 |
| **바인딩** | "이 도메인으로 요청이 오면 이 인증서를 제시해라"는 설정 |

인증서가 서버에 있더라도 바인딩이 없으면 서버는 엉뚱한 기본 인증서를 내놓는다.

### 일반적인 수정 절차

1. **`example.com`을 커버하는 인증서를 발급받는다.**
   - Let's Encrypt (무료, `certbot` 사용) — 가장 흔한 방법
   - 호스팅 플랫폼의 managed certificate (플랫폼이 자동 발급·갱신)
   - 직접 구매한 인증서 (.pem / .pfx)

2. **인증서를 서버/플랫폼에 바인딩한다.**
   - Nginx: `ssl_certificate`, `ssl_certificate_key` 지시어로 지정
   - Apache: `SSLCertificateFile`, `SSLCertificateKeyFile`로 지정
   - 클라우드 플랫폼: 커스텀 도메인 설정 화면에서 인증서 선택 후 바인딩

### Nginx 예시

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ...
}
```

certbot을 쓴다면:

```bash
certbot --nginx -d example.com
# certbot이 nginx 설정까지 자동으로 잡아준다
```

### Azure 예시 (App Service / Container Apps)

포털에서:

```
앱 리소스 → Custom domains
  → example.com 옆 "No binding" 클릭
  → Add binding
  → Source = Create managed certificate
  → TLS/SSL type = SNI SSL
  → Add
```

**App Service:**

```bash
# 1. managed cert 발급
az webapp config ssl create \
  --resource-group <resource-group> \
  --name <app-name> \
  --hostname example.com

# 2. thumbprint 확인
az webapp config ssl list --resource-group <resource-group> \
  --query "[?subjectName=='example.com'].thumbprint" -o tsv

# 3. 바인딩 (SNI SSL)
az webapp config ssl bind \
  --resource-group <resource-group> \
  --name <app-name> \
  --certificate-thumbprint <thumbprint> \
  --ssl-type SNI
```

**Container Apps** (인증서는 앱이 아니라 Environment에 붙는다):

```bash
az containerapp hostname add  -g <resource-group> -n <app-name> --hostname example.com
az containerapp hostname bind -g <resource-group> -n <app-name> --hostname example.com \
  --environment <environment> --validation-method CNAME
```

### 확인

어느 플랫폼이든 확인 방법은 동일하다.

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -ext subjectAltName
```

출력에서 SAN에 `example.com`이 들어있으면 성공이다. `ERR_TLS_CERT_ALTNAME_INVALID`는 더 이상 뜨지 않는다.

## 주의사항

1. **TLS를 실제로 종료하는 곳에 바인딩해야 한다.** 리버스 프록시(Nginx, Caddy, 로드밸런서)가 앞에 있으면 인증서는 프록시에 걸려야 한다. 백엔드 앱 서버에 걸어봐야 의미가 없다.

2. **DNS 조건이 먼저 맞아야 인증서 발급이 된다.**
   - 루트 도메인 (`example.com`) → A 레코드 필요
   - 서브도메인 (`www.example.com`) → CNAME 가능
   - CAA 레코드가 있다면 사용하는 CA를 허용해야 함 (예: `0 issue letsencrypt.org`)

3. **와일드카드 인증서가 아니면 호스트네임별로 따로 발급해야 한다.** `example.com`과 `www.example.com`은 별개다.

4. **`rejectUnauthorized: false`로 퉁치지 말자.** 에러는 사라지지만 TLS 검증을 통째로 꺼버리는 거다. 웹훅 채널이 위조 공격에 노출된다.

5. **자동 갱신은 조건부다.** Let's Encrypt든 managed cert든, DNS 레코드가 살아있고 도메인이 서버를 가리키고 있어야 갱신이 된다. A/CNAME 레코드가 바뀌면 갱신 실패한다.

---

## 3줄 요약

1. `ERR_TLS_CERT_ALTNAME_INVALID`는 서버 인증서의 SAN 목록에 접속하려는 호스트명이 없을 때 발생한다 — 플랫폼 불문, 도메인을 라우팅만 해놓고 인증서를 바인딩하지 않으면 생기는 일이다.
2. DNS 라우팅(도메인 → 서버로 트래픽 전달)과 TLS 신원(서버가 그 도메인임을 증명)은 완전히 독립적이다 — 트래픽이 도착한다고 인증서가 생기는 게 아니다.
3. 수정은 해당 도메인의 인증서를 발급받고 서버에 바인딩하는 것으로 끝난다; 핸드셰이크 전에 실패하는 문제이므로 애플리케이션 코드를 디버깅해봐야 의미없다.
