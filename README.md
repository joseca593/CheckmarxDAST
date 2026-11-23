# Vista de seguridad · Pago de préstamos QR Red Externa

## Arquitectura

```mermaid
architecture-beta
    group browser(internet)[Navegador del usuario]
    service recaptcha(cloud)[1 Google reCAPTCHA Enterprise] in browser
    service spa(server)[Cliente SPA Pago QR] in browser
    service ciamjwks(disk)[SaaS CIAM JWT y JWKS] in browser

    group cf(cloud)[Cloudflare borde publico]
    service cfedge(internet)[3 WAF OWASP TLS Rate limit Header secreto] in cf
    service cfws(internet)[W1 Ruta WebSocket wss] in cf
    service cfrest(internet)[4 Ruta API REST] in cf
    service cfjwks(internet)[2 K5 Ruta JWKS publico] in cf

    group core(cloud)[AWS Cuenta core]
    service apiws(server)[API Gateway WebSocket] in core
    service conn(server)[W3 Salida connections IAM] in core
    service awswaf(server)[5 AWS WAF regional] in core
    service apirest(server)[API Gateway REST] in core
    service authz(server)[6 W2 Lambda authorizer] in core
    service negocio(server)[7 Servicios de negocio ECS Fargate] in core
    service decrypt(server)[8 Servicio Decrypt JWE ECS Fargate] in core

    group seg(cloud)[AWS Cuenta seguridad]
    service pipeline(server)[K1 K2 Pipeline JWK Cuenta DevOps] in seg
    service secrets(disk)[9 Secrets Manager llave privada] in seg
    service kms(disk)[AWS KMS] in seg
    service s3seg(database)[S3 llave publica] in seg

    group integ(cloud)[AWS Cuenta integraciones]
    service s3int(database)[K3 S3 JWKS publico] in integ
    service cfront(internet)[K4 CloudFront OAC jwks json] in integ

    recaptcha:B --> T:spa
    ciamjwks:T --> B:spa
    spa:R --> L:cfedge
    cfedge:T --> B:cfrest
    cfrest:T -- B:cfws
    cfedge:B -- T:cfjwks

    cfws:R --> L:apiws
    conn:L --> R:apiws
    authz:T --> B:apiws
    cfrest:R --> L:awswaf
    awswaf:R --> L:apirest
    apirest:R --> L:authz
    apirest:B --> T:negocio
    decrypt:L --> R:negocio

    secrets:L --> R:decrypt
    pipeline:B --> T:secrets
    kms:L --> R:secrets
    pipeline:R --> L:s3seg
    s3seg:R --> L:s3int
    s3int:B --> T:cfront
```

> Los números de cada ícono corresponden a los pasos de las secuencias de abajo. La validación del JWT (JWKS del CIAM) y del token reCAPTCHA en el Lambda authorizer, y la publicación K5 de CloudFront hacia Cloudflare, se detallan en las secuencias y la tabla.

## Secuencia de una petición

```mermaid
sequenceDiagram
    autonumber
    participant SPA as Cliente SPA
    participant RC as reCAPTCHA Enterprise
    participant CF as Cloudflare
    participant WAF as AWS WAF regional
    participant API as API Gateway REST
    participant AUTH as Lambda authorizer
    participant SVC as Servicios de negocio
    participant DEC as Servicio Decrypt
    participant SM as Secrets Manager

    SPA->>RC: Obtiene token reCAPTCHA
    SPA->>CF: Descarga JWKS público de BIF
    SPA->>SPA: Cifra payload JWE con llave pública
    SPA->>CF: HTTPS con Bearer JWT + token reCAPTCHA
    CF->>WAF: WAF, bots, DDoS, rate limit + header secreto
    WAF->>API: Solo tráfico con header secreto
    API->>AUTH: Valida JWT con JWKS del CIAM y reCAPTCHA
    AUTH-->>API: Allow o Deny
    API->>SVC: Petición autorizada por VPC Link y ALB
    SVC->>DEC: Pide descifrar el payload
    DEC->>SM: GetSecretValue con task role por VPC endpoint
    SM-->>DEC: Llave privada
    DEC-->>SVC: Payload descifrado
```

## Canal WebSocket · confirmación del pago

```mermaid
sequenceDiagram
    participant SPA as Cliente SPA
    participant CF as Cloudflare
    participant WS as API Gateway WebSocket
    participant AUTH as Lambda authorizer
    participant SVC as Servicio backend

    SPA->>CF: W1 abre wss con su JWT
    CF->>WS: $connect
    WS->>AUTH: W2 valida JWT y Origin
    AUTH-->>WS: Allow o Deny
    SVC->>WS: W3 PostToConnection firmado con IAM
    WS-->>SPA: Confirmación del pago
```

## Ciclo de las llaves JWK

| Paso | Descripción |
|------|-------------|
| K1 | El Pipeline JWK genera el par de llaves y guarda la llave privada en Secrets Manager, cifrada con KMS |
| K2 | Guarda la llave pública en el S3 de la cuenta de seguridad |
| K3 | La llave pública se sincroniza al S3 de la cuenta de integraciones |
| K4 | CloudFront la sirve desde S3 con OAC como JWKS público |
| K5 | Cloudflare publica el JWKS para que lo descargue el navegador |

> **Regla clave:** la llave privada nunca sale de Secrets Manager. Por Cloudflare solo se expone el JWKS público. Solo el task role del Servicio Decrypt puede leer la llave privada, y KMS la mantiene cifrada.
