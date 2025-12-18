- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
Yep — this screenshot confirms the **exact situation**:

- Deephaven is failing inside the **container** with: `libjvm.so: cannot open shared object file`
    
- You ran `java --version` and it shows Java 17 ✅ — but **that is on the WEnix host**, not inside the container.
    

So your image `deephaven-local:1.0` still **doesn’t have Java installed / configured**.

---

## 1) First prove it: check Java _inside the container_

Run:

```bash
docker run --rm -it --entrypoint bash deephaven-local:1.0 -lc "java -version || echo NO_JAVA"
```

If it prints `NO_JAVA`, that’s the proof.

---

## 2) Fix the Dockerfile: install JDK + set JAVA_HOME + LD_LIBRARY_PATH

Use **Java 17** since UBI clearly supports it in your environment:

```dockerfile
FROM registry.access.redhat.com/ubi8/ubi

RUN dnf install -y \
      python39 \
      python39-pip \
      java-17-openjdk-headless \
      gcc \
      gcc-c++ \
      make \
      libstdc++ \
      glibc-langpack-en \
      openssl \
    && dnf clean all

ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk
ENV PATH="${JAVA_HOME}/bin:${PATH}"
ENV LD_LIBRARY_PATH="${JAVA_HOME}/lib/server:${JAVA_HOME}/lib:${LD_LIBRARY_PATH}"

WORKDIR /app
COPY wheels/ /wheels/

RUN pip3.9 install --no-index --find-links=/wheels deephaven-server

EXPOSE 10000
CMD ["/usr/local/bin/deephaven", "server", "--host", "0.0.0.0", "--port", "10000"]
```

That `LD_LIBRARY_PATH` line is important because `jpy`/JVM bootstrap often needs help finding `libjvm.so`.

---

## 3) Rebuild with a NEW tag (avoid accidentally running old image)

```bash
docker build --no-cache -t deephaven-local:2.0 .
```

---

## 4) Verify JVM bits inside the new image

```bash
docker run --rm -it --entrypoint bash deephaven-local:2.0 -lc '
java -version
echo "JAVA_HOME=$JAVA_HOME"
ls -l $JAVA_HOME/lib/server/libjvm.so
'
```

You must see `libjvm.so` listed.

---

## 5) Run Deephaven

```bash
docker run --rm --name deephaven -p 10000:10000 deephaven-local:2.0
```

---

### Why your `java --version` didn’t help

Because it was executed on **WEnix host shell**. Containers don’t “inherit” host-installed Java unless the image includes it.

---

If you paste the output of step (4) (the `ls -l ... libjvm.so` line), I’ll confirm you’re 100% good before you wire PSK/origins back in.