**# Vista de seguridad · Pago de préstamos QR Red Externa**

**## Arquitectura**

\`\`\`mermaid
architecture-beta
    group browser(internet)[Navegador del usuario]
    service spa(server)[Cliente SPA Modulo Pago QR] in browser
    service recaptcha(cloud)[Google reCAPTCHA Enterprise] in browser

    group ciam(cloud)[SaaS CIAM]
    service ciamauth(server)[AuthN valida OTP] in ciam
    service ciamjwt(disk)[Genera JWT firmado] in ciam
    service ciamjwks(disk)[JWKS del CIAM] in ciam

    group cf(cloud)[Cloudflare borde publico]
    service cfedge(internet)[WAF OWASP TLS Rate limit Header secreto] in cf
    service cfrest(internet)[Ruta API REST] in cf
    service cfws(internet)[Ruta WebSocket wss] in cf
    service cfjwks(internet)[Ruta JWKS publico] in cf

    group aws(cloud)[AWS Cloud BIF]

    group core(cloud)[Cuenta core] in aws
    service awswaf(server)[AWS WAF regional] in core
    service apirest(server)[API Gateway REST] in core
    service apiws(server)[API Gateway WebSocket] in core
    service authz(server)[Lambda authorizer REST y WS] in core
    service conn(server)[Salida connections IAM] in core

    group ecs(server)[Cluster ECS Fargate 3 AZ] in core
    service negocio(server)[Servicios de negocio] in ecs
    service decrypt(server)[Servicio Decrypt JWE] in ecs

    group devops(cloud)[Cuenta DevOps] in aws
    service pipeline(server)[Pipeline JWK] in devops

    group seg(cloud)[Cuenta seguridad] in aws
    service secrets(disk)[Secrets Manager llave privada] in seg
    service kms(disk)[AWS KMS] in seg
    service s3seg(database)[S3 llave publica] in seg

    group integ(cloud)[Cuenta integraciones] in aws
    service s3int(database)[S3 JWKS publico] in integ
    service cfront(internet)[CloudFront OAC jwks json] in integ

    %% Autenticacion y reCAPTCHA
    ciamauth:R --> L:ciamjwt
    ciamjwt:T --> B:spa
    spa:B --> T:recaptcha

    %% Peticion REST (pasos 3 a 9)
    spa:R --> L:cfedge
    cfedge:R --> L:cfrest
    cfedge:B --> T:cfws
    cfrest:R --> L:awswaf
    awswaf:R --> L:apirest
    apirest:R --> L:authz
    recaptcha:R --> L:authz
    ciamjwks:R --> L:authz
    apirest:B --> T:negocio
    decrypt:L --> R:negocio
    secrets:L --> R:decrypt

    %% Canal WebSocket (W1 a W3)
    cfws:R --> L:apiws
    authz:T --> B:apiws
    conn:L --> R:apiws

    %% Ciclo de llaves JWK (K1 a K5)
    pipeline:B --> T:secrets
    pipeline:R --> T:s3seg
    kms:L --> R:secrets
    s3seg:R --> L:s3int
    s3int:B --> T:cfront
    cfront:L --> R:cfjwks
    cfjwks:L --> R:spa
\`\`\`

**## Secuencia de una petición**

\`\`\`mermaid
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
\`\`\`

**## Canal WebSocket · confirmación del pago**

\`\`\`mermaid
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
\`\`\`

**## Ciclo de las llaves JWK**

\| Paso | Descripción |
\|------|-------------|
\| K1 | El Pipeline JWK genera el par de llaves y guarda la llave privada en Secrets Manager, cifrada con KMS |
\| K2 | Guarda la llave pública en el S3 de la cuenta de seguridad |
\| K3 | La llave pública se sincroniza al S3 de la cuenta de integraciones |
\| K4 | CloudFront la sirve desde S3 con OAC como JWKS público |
\| K5 | Cloudflare publica el JWKS para que lo descargue el navegador |

\> **\*\*Regla clave:\*\*** la llave privada nunca sale de Secrets Manager. Por Cloudflare solo se expone el JWKS público. Solo el task role del Servicio Decrypt puede leer la llave privada, y KMS la mantiene cifrada.
