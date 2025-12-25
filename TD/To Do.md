Absolutely — below is a **complete end-to-end Docker Compose setup** that runs **all 5** of your services together:

- ✅ **deephaven**
    
- ✅ **dh-orchestrator**
    
- ✅ **thales-edge** (nginx + correct proxy routes)
    
- ✅ **streaming-ingestion-sdk** (your Spring Boot consumer/ingestion app)
    
- ✅ **bishowcase-backend** (your Spring Boot backend)
    

This is written so teammates can run **one command** and everything comes up on the same network.

---

# Folder layout (recommended)

Create a folder like:

```
stack/
  docker-compose.yml
  .env
  nginx/
    thales-edge.conf
  secrets/
    truststore.jks         (optional)
  configs/
    databruckscfg          (optional)
```

---

# 1) `.env` (central place for runtime values)

Create `stack/.env`:

```env
# Deephaven
DEEPHAVEN_PSK=my-fixed-psk

# Databricks (if needed by ingestion apps)
DATABRICKS_HOST=https://<your-workspace>
DATABRICKS_TOKEN=<your-token>
DATABRICKS_HTTP_PATH=/sql/1.0/warehouses/<id>

# Kafka / Confluent
KAFKA_BOOTSTRAP=<bootstrap:9092>
KAFKA_SECURITY_PROTOCOL=SASL_SSL
KAFKA_SASL_MECHANISM=OAUTHBEARER
KAFKA_SASL_JAAS_CONFIG=<your-jaas-config>
KAFKA_SASL_OAUTHBEARER_TOKEN_ENDPOINT_URL=<token-endpoint>

# Optional - TLS truststore for Kafka
SSL_TRUSTSTORE_LOCATION=/run/secrets/truststore.jks
SSL_TRUSTSTORE_PASSWORD=<password>

# Spring profile knobs (optional)
SPRING_PROFILES_ACTIVE=docker
```

If you don’t want to store tokens in `.env`, you can still pass them at runtime — but this keeps it “one command”.

---

# 2) `docker-compose.yml` (FULL end-to-end)

Create `stack/docker-compose.yml`:

```yaml
services:
  # ---------------------------
  # 1) Deephaven
  # ---------------------------
  deephaven:
    image: deephaven-local:1.3.0
    container_name: deephaven
    environment:
      DEEPHAVEN_AUTH_TYPE: "psk"
      DEEPHAVEN_PSK: "${DEEPHAVEN_PSK}"
      START_OPTS: "-Ddeephaven.application.dir=/app.d"
    ports:
      - "10000:10000"
      - "10001:10001"
    volumes:
      - /shared-resources/deephaven/app.d:/app.d:ro
    networks: [dh-net]

  # ---------------------------
  # 2) dh-orchestrator (talks to Deephaven)
  # ---------------------------
  dh-orchestrator:
    image: dh-orchestrator:1.0
    container_name: dh-orchestrator
    environment:
      SERVER_PORT: "8081"
      DEEPHAVEN_BASE_URL: "http://deephaven:10000"
      DEEPHAVEN_PSK: "${DEEPHAVEN_PSK}"
      SPRING_PROFILES_ACTIVE: "${SPRING_PROFILES_ACTIVE}"
    ports:
      - "8081:8081"
    depends_on:
      - deephaven
    networks: [dh-net]

  # ---------------------------
  # 3) thales-edge (nginx) - browser hits only localhost:4200
  #    nginx proxies Deephaven paths internally to http://deephaven:10000
  # ---------------------------
  thales-edge:
    image: thales-edge:latest
    container_name: thales-edge
    ports:
      - "4200:8080"
    environment:
      DEEPHAVEN_PSK: "${DEEPHAVEN_PSK}"
      SPRING_PROFILES_ACTIVE: "${SPRING_PROFILES_ACTIVE}"
    volumes:
      - ./nginx/thales-edge.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - deephaven
      - dh-orchestrator
      - bishowcase-backend
    networks: [dh-net]

  # ---------------------------
  # 4) BIShowcase backend (Spring Boot)
  # ---------------------------
  bishowcase-backend:
    image: bishowcase-backend:latest
    container_name: bishowcase-backend
    environment:
      SERVER_PORT: "8080"
      SPRING_PROFILES_ACTIVE: "${SPRING_PROFILES_ACTIVE}"

      # If backend calls dh-orchestrator:
      DH_ORCHESTRATOR_BASE_URL: "http://dh-orchestrator:8081"

      # If backend needs Databricks:
      DATABRICKS_HOST: "${DATABRICKS_HOST}"
      DATABRICKS_TOKEN: "${DATABRICKS_TOKEN}"
      DATABRICKS_HTTP_PATH: "${DATABRICKS_HTTP_PATH}"

    ports:
      - "8080:8080"
    depends_on:
      - dh-orchestrator
    networks: [dh-net]

  # ---------------------------
  # 5) streaming-ingestion-sdk app (Spring Boot consumer/ingester)
  # ---------------------------
  streaming-ingestion-sdk:
    image: streaming-ingestion-sdk:latest
    container_name: streaming-ingestion-sdk
    environment:
      SERVER_PORT: "8090"
      SPRING_PROFILES_ACTIVE: "${SPRING_PROFILES_ACTIVE}"

      # Kafka
      KAFKA_BOOTSTRAP: "${KAFKA_BOOTSTRAP}"
      KAFKA_SECURITY_PROTOCOL: "${KAFKA_SECURITY_PROTOCOL}"
      KAFKA_SASL_MECHANISM: "${KAFKA_SASL_MECHANISM}"
      KAFKA_SASL_JAAS_CONFIG: "${KAFKA_SASL_JAAS_CONFIG}"
      KAFKA_SASL_OAUTHBEARER_TOKEN_ENDPOINT_URL: "${KAFKA_SASL_OAUTHBEARER_TOKEN_ENDPOINT_URL}"

      # TLS truststore (optional)
      SSL_TRUSTSTORE_LOCATION: "${SSL_TRUSTSTORE_LOCATION}"
      SSL_TRUSTSTORE_PASSWORD: "${SSL_TRUSTSTORE_PASSWORD}"

      # Databricks
      DATABRICKS_HOST: "${DATABRICKS_HOST}"
      DATABRICKS_TOKEN: "${DATABRICKS_TOKEN}"
      DATABRICKS_HTTP_PATH: "${DATABRICKS_HTTP_PATH}"

    # If it exposes health endpoints:
    ports:
      - "8090:8090"

    # If you want a truststore available as a docker secret:
    secrets:
      - truststore_jks

    depends_on:
      - bishowcase-backend
    networks: [dh-net]

networks:
  dh-net:
    name: dh-net

secrets:
  truststore_jks:
    file: ./secrets/truststore.jks
```

---

# 3) Nginx proxy for thales-edge (critical)

Create `stack/nginx/thales-edge.conf`:

```nginx
server {
  listen 8080;
  server_name _;

  root /opt/app-root/src;
  index index.html;

  location / {
    try_files $uri $uri/ /index.html;
  }

  # Deephaven JS runtime (dh-core.js)
  location /js/ {
    proxy_pass http://deephaven:10000/js/;
    proxy_set_header Host $host;
  }

  # Safe to keep if your code still references /jsapi
  location /jsapi/ {
    proxy_pass http://deephaven:10000/jsapi/;
    proxy_set_header Host $host;
  }

  # Deephaven websocket/session
  location /dh/ {
    proxy_pass http://deephaven:10000/dh/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
  }

  # Deephaven grpc-web endpoints
  location /grpc-web/ {
    proxy_pass http://deephaven:10000/grpc-web/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
  }

  # Some clients hit grpc methods at this prefix
  location /io.deephaven.proto {
    proxy_pass http://deephaven:10000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
  }

  # API routing to your backend(s) (adjust paths to your app)
  location /api/ {
    proxy_pass http://bishowcase-backend:8080/api/;
    proxy_set_header Host $host;
  }

  location /orchestrator/ {
    proxy_pass http://dh-orchestrator:8081/;
    proxy_set_header Host $host;
  }
}
```

---

# 4) What should your Angular Deephaven import path be?

In your Angular `deephaven.service.ts` (production/nginx mode):

✅ Use **same-origin** and load dh-core from `/js/`:

```ts
const coreUrl = '/js/dh-core.js';
this.dh = (await import(/* @vite-ignore */ coreUrl)).default;
this.client = new this.dh.CoreClient(window.location.origin);
```

This ensures the browser only talks to `http://localhost:4200/...` and nginx forwards internally.

---

# 5) Run the whole stack

From inside `stack/`:

```bash
docker compose up -d
docker compose ps
```

Open:

- UI: `http://localhost:4200`
    
- Backend: `http://localhost:8080`
    
- dh-orchestrator: `http://localhost:8081`
    
- Deephaven direct: `http://localhost:10000`
    

---

# 6) Smoke tests (use these to confirm wiring)

### UI can fetch dh-core.js through nginx proxy

```bash
curl -I http://localhost:4200/js/dh-core.js
```

### Orchestrator can reach Deephaven

```bash
docker exec -it dh-orchestrator sh -lc "curl -I http://deephaven:10000/js/dh-core.js | head"
```

### thales-edge nginx can resolve Deephaven and backend

```bash
docker exec -it thales-edge sh -lc "getent hosts deephaven bishowcase-backend dh-orchestrator"
```

---

# 7) Notes for streaming-ingestion-sdk & backend (important)

## A) Kafka auth + truststore

If your ingestion app needs a truststore, you now have two clean options:

- **docker secret** (included above)
    
- or mount a volume
    

If you prefer volume instead of secrets:

```yaml
volumes:
  - /shared-resources/certs/truststore.jks:/opt/certs/truststore.jks:ro
environment:
  SSL_TRUSTSTORE_LOCATION: /opt/certs/truststore.jks
```

## B) Databricks token expiry

Your `.env` uses `DATABRICKS_TOKEN`. If it expires, you update `.env` and restart just the ingestion container:

```bash
docker compose restart streaming-ingestion-sdk
```

(Or use a service principal / PAT rotation later.)

---

# 8) If your apps use different env var names

I used generic env keys like `DATABRICKS_HOST`, `KAFKA_BOOTSTRAP`, etc.  
If your Spring apps expect `spring.kafka.bootstrap-servers` style, you can still pass those as env vars using Spring’s relaxed binding, e.g.:

```yaml
environment:
  SPRING_KAFKA_BOOTSTRAP_SERVERS: "${KAFKA_BOOTSTRAP}"
```

Same for your custom properties.

---

## If you paste two small things, I’ll make this 100% exact to your repo:

1. `application.yml` env keys used by **streaming-ingestion-sdk** (just the kafka + databricks section)
    
2. `application.yml` env keys used by **bishowcase-backend** (just server port + API base paths)
    

I’ll then rewrite the compose with **your exact property names** so you don’t have to translate anything.