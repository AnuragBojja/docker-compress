# Payment Service — Docker Challenges & Solutions

## Overview

This document captures the challenges faced while containerizing the **Payment** service using a multi-stage Docker build on Alpine Linux, and how each was resolved.

---

## Challenge 1: uWSGI Build Failure — Missing C Compiler

### Error
```
Exception: you need a C compiler to build uWSGI
```

### Cause
The base image `python:3.14-alpine3.22` uses Alpine Linux, which does **not** ship with a C compiler (`gcc`) by default. uWSGI is a C extension that must be compiled from source.

### Fix
Install `gcc` before running `pip3 install`:
```dockerfile
RUN apk add --no-cache gcc && \
    pip3 install --prefix=/install -r requirements.txt
```

---

## Challenge 2: Missing Standard C Library Headers (`stdio.h`)

### Error
```
fatal error: stdio.h: No such file or directory
```

### Cause
Alpine uses **musl libc** instead of glibc. The musl C standard library headers (like `stdio.h`) are not included by default — they come from the `musl-dev` package.

> **Note:** Documentation written for RHEL/CentOS won't mention this package because glibc headers are available by default on those systems. On RHEL it is called `python-devel`, on Alpine it is `python3-dev`.

### Fix
Add `musl-dev` to the apk install command:
```dockerfile
RUN apk add --no-cache gcc musl-dev && \
    pip3 install --prefix=/install -r requirements.txt
```

---

## Challenge 3: Missing Linux Kernel Headers (`linux/limits.h`)

### Error
```
fatal error: linux/limits.h: No such file or directory
```

### Cause
uWSGI needs Linux kernel headers to interface with the OS. These are provided by the `linux-headers` package, which is also absent by default in Alpine.

### Fix
Add `linux-headers` to the apk install command:
```dockerfile
RUN apk add --no-cache gcc musl-dev linux-headers && \
    pip3 install --prefix=/install -r requirements.txt
```

This is the **complete set of packages** required to compile uWSGI on Alpine.

---

## Challenge 4: `uwsgi` Executable Not Found at Runtime

### Error
```
exec: "uwsgi": executable file not found in $PATH
```

### Cause
A multi-stage build was used — packages were installed in the **build stage**, but only `/opt/server` (app files) was copied to the final stage. The pip-installed packages and the `uwsgi` binary were left behind in the build stage.

### Fix
Use `--prefix=/install` when installing with pip, then copy the entire `/install` directory to `/usr/local/` in the final stage:
```dockerfile
# Build stage
RUN pip3 install --prefix=/install -r requirements.txt

# Final stage
COPY --from=build /install /usr/local/
```

**Why `--prefix` over `--target`:**

| | `--target` | `--prefix` |
|---|---|---|
| Structure | Flat | Full Python structure |
| Includes binaries (`uwsgi`) | ❌ No | ✅ Yes |
| Copy destination | `/usr/local/lib/python3.14/site-packages/` | `/usr/local/` |
| Simpler COPY | ❌ Long path | ✅ Short path |

`--prefix` preserves the full directory structure and includes binaries like `uwsgi` automatically — making it the correct choice for multi-stage builds.

---

## Challenge 5: `ModuleNotFoundError: No module named 'instana'`

### Error
```
ModuleNotFoundError: No module named 'instana'
unable to load app 0 (mountpoint='') (callable not found or import error)
```

### Cause
`--target` was used instead of `--prefix` for pip install. The `--target` flag installs packages in a **flat structure**, so when copied to `/usr/local/`, packages landed in the wrong location:

```
# Where packages landed (wrong)
/usr/local/instana/

# Where Python looks for them (correct)
/usr/local/lib/python3.14/site-packages/instana/
```

### Fix
Switch from `--target` to `--prefix`:
```dockerfile
# Wrong
RUN pip3 install --target /packages -r requirements.txt
COPY --from=build /packages /usr/local/   # lands in wrong place

# Correct
RUN pip3 install --prefix=/install -r requirements.txt
COPY --from=build /install /usr/local/    # correct structure preserved
```

---

## Challenge 6: Container Exiting — `cannot setgid() as non-root user`

### Error
```
cannot setgid() as non-root user
```

### Cause
`payment.ini` had `uid` and `gid` set to `1001` (matching the bare-metal server's user). The Dockerfile was running as a non-root `roboshop` user via `USER roboshop`. uWSGI tried to call `setgid()` internally which requires root privileges — causing the crash.

On Alpine, system users created without explicit IDs get auto-assigned UIDs starting from `100`, not `1001`:
```dockerfile
# Alpine assigns uid=100, gid=101 — mismatches payment.ini uid/gid=1001
RUN addgroup -S roboshop && adduser -S roboshop -G roboshop
```

### Fix
Explicitly set `uid` and `gid` to `1001` to match `payment.ini`:
```dockerfile
RUN addgroup -S -g 1001 roboshop && adduser -S -u 1001 roboshop -G roboshop
```

This aligns the container user with the application config, keeping the container non-root while avoiding the `setgid()` conflict.

---

## Final Dockerfile

```dockerfile
FROM python:3.14-alpine3.22 AS build
WORKDIR /opt/server
RUN apk add --no-cache gcc python3-dev musl-dev linux-headers
COPY requirements.txt .
RUN pip3 install --prefix=/install -r requirements.txt


FROM python:3.14-alpine3.22
EXPOSE 8080
WORKDIR /opt/server
COPY --from=build /install /usr/local/
RUN addgroup -S -g 1001 roboshop && adduser -S -u 1001 roboshop -G roboshop
COPY --chown=roboshop:roboshop *.py .
COPY --chown=roboshop:roboshop payment.ini .
ENV CART_HOST=cart \
    CART_PORT=8080 \
    USER_HOST=user \
    USER_PORT=8080 \
    AMQP_HOST=rabbitmq \
    AMQP_USER=roboshop \
    AMQP_PASS=roboshop123
USER roboshop
ENTRYPOINT ["uwsgi"]
CMD ["--ini", "payment.ini"]
```

---

## Summary Table

| # | Challenge | Root Cause | Fix |
|---|-----------|------------|-----|
| 1 | No C compiler | Alpine has no `gcc` by default | `apk add gcc` |
| 2 | `stdio.h` not found | Alpine uses musl libc, not glibc | `apk add musl-dev` |
| 3 | `linux/limits.h` not found | Kernel headers missing on Alpine | `apk add linux-headers` |
| 4 | `uwsgi` not found at runtime | Multi-stage build didn't copy pip packages | `pip3 install --prefix=/install` + `COPY --from=build /install /usr/local/` |
| 5 | `ModuleNotFoundError: instana` | `--target` copies flat structure to wrong location | Switch to `--prefix` which preserves full Python directory structure |
| 6 | `cannot setgid()` as non-root | UID/GID mismatch — Alpine auto-assigns `100/101`, `payment.ini` expects `1001` | Set explicit `uid/gid 1001` when creating user |

---

## Key Learnings

- Alpine Linux is minimal by design — C compilation requires `gcc`, `musl-dev`, and `linux-headers`.
- `musl-dev` and `linux-headers` are **build-time only** — Alpine's base image already includes the musl runtime.
- Use `pip3 install --prefix=/install` over `--target` in multi-stage builds — it preserves the full directory structure and includes binaries.
- Multi-stage builds require **explicit copying** of all runtime artifacts, not just app source files.
- Container user identity should be managed by Docker (`USER` directive), with explicit `uid/gid` set to match application config.
- Documentation written for RHEL/CentOS may use different package names (e.g., `python-devel` → `python3-dev` on Alpine).
- Use `--chown` flag in `COPY` instead of a separate `RUN chown` — sets ownership in one step without an extra layer.
- `ENTRYPOINT` = runtime executable (fixed), `CMD` = arguments (overridable).