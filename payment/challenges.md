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
RUN apk add --no-cache gcc && pip3 install -r requirements.txt
```

---

## Challenge 2: Missing Standard C Library Headers (`stdio.h`)

### Error
```
fatal error: stdio.h: No such file or directory
```

### Cause
Alpine uses **musl libc** instead of glibc. The musl C standard library headers (like `stdio.h`) are not included by default — they come from the `musl-dev` package.

### Fix
Add `musl-dev` to the apk install command:
```dockerfile
RUN apk add --no-cache gcc musl-dev && pip3 install -r requirements.txt
```

> **Note:** Documentation written for RHEL/CentOS won't mention this package because glibc headers are available by default on those systems.

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
RUN apk add --no-cache gcc musl-dev linux-headers && pip3 install -r requirements.txt
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
Explicitly copy /usr/local from the build stage:
```dockerfile
COPY --from=build /usr/local /usr/local
```

---

## Challenge 5: Container Exiting — `cannot setgid() as non-root user`

### Error
```
cannot setgid() as non-root user
```

### Cause
`payment.ini` had `uid` and `gid` set to `1001` (matching the bare-metal server's user). The Dockerfile was running the container as a non-root `roboshop` user via `USER roboshop`. uWSGI tried to call `setgid()` internally, which requires root privileges — causing the crash.

### Fix
Explicitly set the `uid` and `gid` to `1001` when creating the user in the Dockerfile, so it matches what `payment.ini` expects:

```dockerfile
RUN addgroup -S -g 1001 roboshop && adduser -S -u 1001 -G roboshop roboshop
```

This aligns the container user with the application config, keeping the container non-root while avoiding the `setgid()` conflict.

---

## Summary Table

| # | Challenge | Root Cause | Fix |
|---|-----------|------------|-----|
| 1 | No C compiler | Alpine has no `gcc` by default | `apk add gcc` |
| 2 | `stdio.h` not found | Alpine uses musl libc, not glibc | `apk add musl-dev` |
| 3 | `linux/limits.h` not found | Kernel headers missing on Alpine | `apk add linux-headers` |
| 4 | `uwsgi` not found at runtime | Multi-stage build didn't copy pip packages | `COPY --from=build` site-packages and binary |
| 5 | `cannot setgid()` as non-root | UID/GID mismatch between container user and `payment.ini` | Set explicit `uid/gid 1001` when creating user |

---

## Key Learnings

- Alpine Linux is minimal by design — C compilation requires `gcc`, `musl-dev`, and `linux-headers`.
- Multi-stage builds require **explicit copying** of all runtime artifacts, not just app source files.
- Container user identity should be managed by Docker (`USER` directive), with application config aligned to match.
- Documentation written for RHEL/CentOS may use different package names (e.g., `python-devel` → `python3-dev` on Alpine).