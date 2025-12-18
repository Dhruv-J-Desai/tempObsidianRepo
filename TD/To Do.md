- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
Yep — that `jvm.log` tail is showing something _very specific_:

- `witness='java/nio/charset/UnsupportedCharsetException'`
    
- plus a `SocketTimeoutException`
    

The **UnsupportedCharsetException** is the bigger red flag — it almost always happens in minimal Linux images when **locale / LANG / LC_ALL** are not set correctly (or set to something weird), and Java tries to use a charset name that doesn’t exist → Deephaven/JVM aborts during bootstrap.

### Fix: force UTF-8 locale + force Java encoding

Update your Dockerfile like this (keep everything else same):

```dockerfile
FROM registry.access.redhat.com/ubi8/ubi

RUN dnf install -y \
      python39 python39-pip \
      java-17-openjdk-headless \
      gcc gcc-c++ make libstdc++ \
      glibc-langpack-en \
      openssl \
    && dnf clean all

# Force a sane UTF-8 locale inside the container
ENV LANG=en_US.UTF-8
ENV LC_ALL=en_US.UTF-8

# Force Java to use UTF-8 encodings (prevents UnsupportedCharset issues)
ENV JAVA_TOOL_OPTIONS="-Dfile.encoding=UTF-8 -Dsun.jnu.encoding=UTF-8"

WORKDIR /app
COPY wheels/ /wheels/
RUN pip3.9 install --no-index --find-links=/wheels deephaven-server

EXPOSE 10000
CMD ["deephaven", "server", "--host", "0.0.0.0", "--port", "10000"]
```

Then rebuild + run:

```bash
docker build --no-cache -t deephaven-local:1.2.0 .
docker rm -f deephaven 2>/dev/null || true
docker run -d --name deephaven -p 10000:10000 \
  -e DEEPHAVEN_AUTH_TYPE=psk -e DEEPHAVEN_PSK=my-fixed-psk \
  deephaven-local:1.2.0
docker logs -f deephaven
```

---

## Quick verification inside the image (before running Deephaven)

This confirms your locale/encoding is correct:

```bash
docker run --rm -it deephaven-local:1.2.0 bash -lc '
locale;
python3.9 -c "import sys; print(sys.getdefaultencoding())";
java -XshowSettings:properties -version 2>&1 | egrep "file.encoding|sun.jnu.encoding|user.language|user.country"
'
```

You want to see:

- `LANG=en_US.UTF-8`
    
- `file.encoding = UTF-8`
    
- `sun.jnu.encoding = UTF-8`
    

---

### About the SocketTimeoutException

That usually means Deephaven (or a dependency) tried to reach something external and timed out. It _shouldn’t_ crash the server by itself — but once we fix the charset issue, we’ll see if the timeout still matters.

If it still aborts after the UTF-8 fix, paste:

```bash
docker logs deephaven | tail -n 200
```

and we’ll address the next blocker.