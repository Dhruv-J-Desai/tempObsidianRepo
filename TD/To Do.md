Here’s a **complete `docker-compose.yml`** that:

1. Starts **DeepHaven** (which generates a PSK and prints it in logs)
    
2. A **psk-catcher** container reads DeepHaven logs, extracts the PSK, writes it to a shared volume
    
3. **dh-orchestrator** waits for that file, exports `DEEPHAVEN_PSK`, then starts
    
4. **thales-edge-ui** (Angular/nginx) starts normally (and you can optionally make it wait too)
    

> Important: this works best when you run Docker on Linux/WEnix (or Docker Desktop). The `psk-catcher` mounts the Docker socket.

```yaml
name: thales-stack

services:
  # -----------------------
  # 1) DeepHaven
  # -----------------------
  deephaven:
    image: deephaven-local:1.2.0
    container_name: deephaven
    ports:
      - "10000:10000"
    environment:
      DEEPHAVEN_AUTH_TYPE: psk
      # If you want to FORCE a fixed PSK, uncomment the next line:
      # DEEPHAVEN_PSK: "my-fixed-psk"
      START_OPTS: "-Ddeephaven.application.dir=/app.d"
      # If you use CORS:
      # DEEPHAVEN_SERVER_ALLOWED_ORIGINS: "http://localhost:4200,http://localhost:8080"
    volumes:
      - dh-shared:/shared
      - ./deephaven/app.d:/app.d:ro
    networks: [tdsnet]

  # -----------------------
  # 2) PSK catcher (reads deephaven logs, writes /shared/deephaven.psk)
  # -----------------------
  psk-catcher:
    image: docker:cli
    container_name: psk-catcher
    depends_on:
      - deephaven
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - dh-shared:/shared
    networks: [tdsnet]
    command: >
      sh -lc '
        rm -f /shared/deephaven.psk;
        echo "Waiting for DeepHaven PSK in logs...";
        while true; do
          psk=$(docker logs deephaven 2>/dev/null | sed -n "s/.*[?&]psk=\\([^[:space:]]*\\).*/\\1/p" | tail -n 1);
          if [ -n "$psk" ]; then
            echo -n "$psk" > /shared/deephaven.psk;
            echo "Captured PSK: $psk";
            break;
          fi
          sleep 1;
        done
        tail -f /dev/null
      '

  # -----------------------
  # 3) dh-orchestrator (waits for PSK file, then starts)
  # -----------------------
  dh-orchestrator:
    build:
      context: ./dh-orchestrator
      dockerfile: Dockerfile
    container_name: dh-orchestrator
    depends_on:
      - psk-catcher
    ports:
      - "8081:8081"  # change if your orchestrator runs on a different port
    environment:
      DEEPHAVEN_HOST: deephaven
      DEEPHAVEN_PORT: 10000
      # DEEPHAVEN_PSK will be exported at runtime from /shared/deephaven.psk
    volumes:
      - dh-shared:/shared
    networks: [tdsnet]
    entrypoint: >
      sh -lc '
        echo "Waiting for /shared/deephaven.psk...";
        while [ ! -s /shared/deephaven.psk ]; do sleep 1; done
        export DEEPHAVEN_PSK="$(cat /shared/deephaven.psk)"
        echo "Using DEEPHAVEN_PSK=$DEEPHAVEN_PSK"
        exec /app/start.sh
      '

  # -----------------------
  # 4) thales-edge UI (Angular served by nginx)
  # -----------------------
  thales-edge-ui:
    build:
      context: ./thales-edge-ui
      dockerfile: Dockerfile
    container_name: thales-edge-ui
    depends_on:
      - dh-orchestrator
    ports:
      - "4200:80"
    networks: [tdsnet]
    # If your UI truly must wait for PSK file too, you can do:
    # volumes:
    #   - dh-shared:/shared
    # entrypoint: >
    #   sh -lc '
    #     while [ ! -s /shared/deephaven.psk ]; do sleep 1; done
    #     exec nginx -g "daemon off;"
    #   '

networks:
  tdsnet:
    driver: bridge

volumes:
  dh-shared:
```

### What you must adjust (2 things)

1. **`dh-orchestrator` start command**  
    I put `exec /app/start.sh` — change it to whatever your container uses (examples):
    
    - `exec python orchestrator.py`
        
    - `exec java -jar app.jar`
        
    - `exec /usr/local/bin/start-orchestrator`
        
2. Make sure these build folders exist:
    

- `./dh-orchestrator/Dockerfile`
    
- `./thales-edge-ui/Dockerfile`
    

---

### One important point about the UI

If **thales-edge-ui runs in the browser**, it usually **should not talk to DeepHaven directly** (because then the PSK must be in the browser, which is messy/insecure). Best pattern is:

**UI → dh-orchestrator → DeepHaven**

If your UI is currently calling DeepHaven directly, tell me how (what URL it hits), and I’ll give you the cleanest way to route it through `dh-orchestrator` so the UI never needs the PSK.