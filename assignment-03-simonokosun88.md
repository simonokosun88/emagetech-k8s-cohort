# Assignment 03 — simonokosun88

**GitHub username:** simonokosun88
**Date completed:** 2026-07-23
**Git SHA of submitted app:** `sha256:a2bdec41eec6e85ca703a43b0dc661bc2d3f72fd0ccca3cdd44173fb824fa6a5`

## 1. Size comparison table

| Variant            | Size  | Layers | Stop time | Exit code |
|--------------------|-------|--------|-----------|-----------|
| `cohort-greet:naive` | 1.63 GB | 19     | 5.144 s     | 137         |
| `cohort-greet:multi` | 259 MB | 21     | 0.231 s     | 0         |

(Layers = output of `docker image history <tag> | wc -l` minus 1 for the header.)

## 2. Final image digest

`sha256:a2bdec41eec6e85ca703a43b0dc661bc2d3f72fd0ccca3cdd44173fb824fa6a5`

## 3. Answers to the 7 questions

**Q1 — naive size + stop behaviour + why:** `docker image ls cohort-greet:naive --format 'naive  | {{.Size}}' naive  |` 

`1.63GB`

Stop time - `cpu 5.144 total`

ExitCode=`137`

The CMD line of the Dockerfile in the shell form (without “” and []) which means that the it is wrapped in `/bin/sh -c` and the shell is PID 1, so docker will send the SIGTERM signal to the shell instead of the app process (gunicorn -b). The 137 ExitCode means that no app process used SIGTERM so the container was forcefully killed after the 5 seconds timeout.

**Q2 — build output, CACHED vs rebuilt:** 
```
simonokosun@Simons-MacBook-Air assignment-03 % docker image build -t cohort-greet:multi . 2>&1 | grep -E "CACHED|RUN pip"

#3 CACHED
#8 [build 5/5] RUN pip install --no-cache-dir -r requirements.txt
#8 CACHED
#9 CACHED
#10 CACHED
#11 CACHED
#12 CACHED
#13 CACHED
```
All the layers before `COPY app.py` were cached because there was no changes in the Dockerfile lines that created those layers. Every layer after `COPY app.py` was rebuilt because we made a change to the app.py file (appended `"# trivial edit $(date)”` ).

**Q3 — new stop time/exit + which change:** 
`0.01s user 0.01s system 9%` 
Stop time = `cpu 0.231 total`
ExitCode=`0`

Dokerfile change: The CMD line from shell form to exec form, `[“gunicorn", "-b", "0.0.0.0:8080", "app:app”]`. Because the gunicorn is PID 1, it will receive the SIGTERM signal which leads to application the shutting down gracefully and with exit code 0.
**Q4 — size reduction breakdown:** 
```
IMAGE                ID             DISK USAGE   CONTENT SIZE EXTRA
cohort-greet:multi   bd2e2580baab        259MB         55.6MB    U   
cohort-greet:naive   8fb39d92d79f       1.63GB          406MB    U   
```
Image shrunk by 84%
Dockerfile changes:
| Layer            | cohort-greet:naive  | cohort-greet:multi  | 
|--------------------|-------|--------|
| Base image: used `python:3.11 slim` to replace `python:3.11` | `python:3.11: # debian.sh --arch 'arm64' out/ 'trixie' '@1…   156MB`    | `python:3.11-slim: # debian.sh --arch 'arm64' out/ 'trixie' '@1…   109MB`
| Multi-stage build: `COPY --from=build /opt/venv /opt/venv` | The build tools and dependencies of `pip install -r requirements.txt` are inside the final image  | Uses `--no-cache-dir -r flags` to remove downloaded package after installation. Only `/opt/venv` is copied into runtime image     | 
| COPY | `COPY . .` includes everything inside build context ( but .dockerignore was used to exclude unnecessary files): `221 KB`  | `COPY app.py .` Only copies the app file needed: 12 KB    | 
| Switch `CMD` from shell form to exec form | `/bin/sh` is PID 1   | `gunicorn` is PID 1 and receives SIGTERM to handle shutdown    |




**Q5 — cache-mount timings + CI relevance:** 
```
#16 DONE 0.7s
docker image build --no-cache -t cohort-greet:multi .  
0.11s user 0.11s system 2% cpu 9.685 total
#16 DONE 0.7s
docker image build --no-cache -t cohort-greet:multi .  
0.10s user 0.08s system 3% cpu 5.060 total
```

About 4.6 seconds was saved with the warm cache mount build

In a CI pipeline, with real projects, the requirements.txt file will likely have huge dependencies in several hundreds of MBs and GBs. Using cold build where all dependencies will be downloaded may take several minutes per build as against just reading from remote cache mount. But if the runners are ephemeral, the layer cache can be cold.

**Q6 — secret marker + what `ARG` would leak:** 
```
cat: /where-token-was-used
No such file or directory
no leak
```
Even though `ARG PYPI_TOKEN` is a build time variable, it would have been stored and visible in the `docker image history` of the image. So if anyone pulls the image, they would easily view `PYPI_TOKEN`.  

**Q7 — tag vs digest for k8s manifest:** 
`sha256:a2bdec41eec6e85ca703a43b0dc661bc2d3f72fd0ccca3cdd44173fb824fa6a5`

semver is a mutable tag thus it can be retagged without any notice which can be unsuitable for production. A git SHA is also mutable. However, the digest is a unique identifier that is directly related to the image manifest itself thus it is immutable. 
So for a production Kubernetes manifest, I would use a combination of the semver and the digest so that it looks like this: `cohort-greet:1.7.3@sha256:a2bdec41eec6e85ca703a43b0dc661bc2d3f72fd0ccca3cdd44173fb824fa6a5`. The semver will make it easily readable and digest will be used by the container runtime for identification and pulling.
If the security team mandates exact reproducibility, the semver tag will be removed and the digest only will be used because that is the only form that will be no registry compromise and cannot be altered.

## 4. Files

### Final `Dockerfile`
```dockerfile
# ── build stage ──
FROM python:3.11.15-slim AS build
RUN python -m venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt

# ── runtime stage ──
FROM python:3.11.15-slim AS runtime
COPY --from=build /opt/venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH
ENV STUDENT_NAME=simon
WORKDIR /app

# TODO: create a non-root user and chown /app

COPY app.py .
RUN useradd --create-home --shell /bin/bash simon-user && \
    chown -R simon-user:simon-user /app

EXPOSE 8080

# TODO: add a HEALTHCHECK that uses python (not curl) to hit /healthz
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD ["python", "-c", "import urllib.request,sys; \
      try: sys.exit(0 if urllib.request.urlopen('http://localhost:8080/healthz').status==200 else 1); \
      except: sys.exit(1)"]

# TODO: USER ... (non-root)
USER simon-user
# TODO: CMD in exec form, gunicorn binding 0.0.0.0:8080
CMD ["gunicorn", "-b", "0.0.0.0:8080", "app:app"] 
```

### `Dockerfile.naive`

```dockerfile
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 8080
CMD gunicorn -b 0.0.0.0:8080 app:app
```

### `Dockerfile.secret`
```dockerfile
# ── build stage ──
FROM python:3.11.15-slim AS build
RUN python -m venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=secret,id=pypi_token \
    TOKEN=$(cat /run/secrets/pypi_token) && \
    TOKEN_PREFIX=$(echo "$TOKEN" | cut -c1-4) && \
    echo "Using token: ${TOKEN_PREFIX}..." && \
    pip install --no-cache-dir -r requirements.txt && \
    echo "$TOKEN_PREFIX" > /where-token-was-used
# ── runtime stage ──
FROM python:3.11.15-slim AS runtime
COPY --from=build /opt/venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH
ENV STUDENT_NAME=simon
WORKDIR /app

# TODO: create a non-root user and chown /app

COPY app.py .
RUN useradd --create-home --shell /bin/bash simon-user && \
    chown -R simon-user:simon-user /app

EXPOSE 8080

# TODO: add a HEALTHCHECK that uses python (not curl) to hit /healthz
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD ["python", "-c", "import urllib.request,sys; \
      try: sys.exit(0 if urllib.request.urlopen('http://localhost:8080/healthz').status==200 else 1); \
      except: sys.exit(1)"]

# TODO: USER ... (non-root)
USER simon-user
# TODO: CMD in exec form, gunicorn binding 0.0.0.0:8080
CMD ["gunicorn", "-b", "0.0.0.0:8080", "app:app"] 
```

### `.dockerignore`
```.git/ 
.gitignore
__pycache__/ 
*.pyc
Dockerfile*
*.md
.env*
Dockerfile.naive
```

## 5. Evidence

For each, paste the command and output. Trim long output to the relevant lines.

- `docker image ls cohort-greet` (all your tags from Part 4.2)
```
simonokosun@Simons-MacBook-Air assignment-03 % docker image ls cohort-greet

IMAGE                 ID             DISK USAGE   CONTENT SIZE   EXTRA
cohort-greet:0.1.0    72a185435106        259MB         55.6MB        
cohort-greet:0.1.0-   72a185435106        259MB         55.6MB        
cohort-greet:git-     72a185435106        259MB         55.6MB        
cohort-greet:multi    72a185435106        259MB         55.6MB        
cohort-greet:naive    53b66aeffb03       1.63GB          405MB 
```
- `docker image history cohort-greet:multi` (truncate long base-image rows)
```
simonokosun@Simons-MacBook-Air assignment-03 % docker image history cohort-greet:multi          
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
72a185435106   3 hours ago   CMD ["gunicorn" "-b" "0.0.0.0:8080" "app:app…   0B        buildkit.dockerfile.v0
<missing>      3 hours ago   USER simon-user                                 0B        buildkit.dockerfile.v0
<missing>      3 hours ago   HEALTHCHECK &{["CMD" "python" "-c" "import u…   0B        buildkit.dockerfile.v0
<missing>      3 hours ago   EXPOSE map[8080/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      3 hours ago   RUN /bin/sh -c useradd --create-home --shell…   77.8kB    buildkit.dockerfile.v0
<missing>      3 hours ago   COPY app.py . # buildkit                        12.3kB    buildkit.dockerfile.v0
<missing>      3 hours ago   WORKDIR /app                                    8.19kB    buildkit.dockerfile.v0
<missing>      3 hours ago   ENV STUDENT_NAME=simon                          0B        buildkit.dockerfile.v0
<missing>      3 hours ago   ENV PATH=/opt/venv/bin:/usr/local/bin:/usr/l…   0B        buildkit.dockerfile.v0
<missing>      3 hours ago   COPY /opt/venv /opt/venv # buildkit             37.2MB    buildkit.dockerfile.v0
<missing>      10 days ago   CMD ["python3"]                                 0B        buildkit.dockerfile.v0
<missing>      10 days ago   RUN /bin/sh -c set -eux;  for src in idle3 p…   16.4kB    buildkit.dockerfile.v0
<missing>      10 days ago   RUN /bin/sh -c set -eux;   savedAptMark="$(a…   52MB      buildkit.dockerfile.v0
<missing>      10 days ago   ENV PYTHON_SHA256=272179ddd9a2e41a0fc8e42e33…   0B        buildkit.dockerfile.v0
<missing>      10 days ago   ENV PYTHON_VERSION=3.11.15                      0B        buildkit.dockerfile.v0
<missing>      10 days ago   ENV GPG_KEY=A035C8C19219BA821ECEA86B64E628F8…   0B        buildkit.dockerfile.v0
<missing>      10 days ago   RUN /bin/sh -c set -eux;  apt-get update;  a…   4.99MB    buildkit.dockerfile.v0
<missing>      10 days ago   ENV LANG=C.UTF-8                                0B        buildkit.dockerfile.v0
<missing>      10 days ago   ENV PATH=/usr/local/bin:/usr/local/sbin:/usr…   0B        buildkit.dockerfile.v0
<missing>      12 days ago   # debian.sh --arch 'arm64' out/ 'trixie' '@1…   109MB     debuerreotype 0.17
```
- `docker container run --rm cohort-greet:secret cat /where-token-was-used`

```
simonokosun@Simons-MacBook-Air assignment-03 % docker container run --rm cohort-greet:secret cat /where-token-was-used
cat: /where-token-was-used: No such file or directory
```
- The "no leak" / "LEAKED" check from Part 3.2
```
simonokosun@Simons-MacBook-Air assignment-03 % docker image history --no-trunc cohort-greet:secret | grep -i "$PYPI_TOKEN" \
pipe> && echo "LEAKED" || echo "no leak"
no leak
```
- `docker container run --rm hadolint/hadolint < Dockerfile` (should be empty)
```
simonokosun@Simons-MacBook-Air assignment-03 % docker container run --rm hadolint/hadolint < Dockerfile
simonokosun@Simons-MacBook-Air assignment-03 % 
```
- The two timing lines from Part 3.1 (cold vs warm cache mount)
```
simonokosun@Simons-MacBook-Air assignment-03 % { time docker image build --no-cache -t cohort-greet:multi . ; } 2>&1 | tail -2
#18 DONE 0.7s
docker image build --no-cache -t cohort-greet:multi .  0.10s user 0.11s system 2% cpu 7.422 total
simonokosun@Simons-MacBook-Air assignment-03 % { time docker image build --no-cache -t cohort-greet:multi . ; } 2>&1 | tail -s2
#16 DONE 0.7s
docker image build --no-cache -t cohort-greet:multi .  0.10s user 0.10s system 3% cpu 5.664 total
```
- (Optional) URL of your pushed image
`ghcr.io/simonokosun88/cohort-greet:0.1.0-7e3e6fc`

## 6. One trade-off I had to make

(2–4 sentences. Pick **one** decision where the slides offered multiple options and you had to choose: alpine vs slim vs distroless, USER 1000 vs `useradd app`, healthcheck via python vs installing curl, etc. Explain why you chose what you chose and what you'd give up by picking the other.)

I created the non-root user with `RUN useradd --create-home --shell /bin/bash simon-user` instead of `USER 1000`, and use `chown` to give the file ownership to simon-user. Even though creating a home directory may add size to the image size, the difference is negligible, when I `docker exec`, the output and log will be more readable.

## 7. One thing I'm still unsure about

(One sentence. Goes to office hours.)

app.py has a variable called `STUDENT_NAME`, do I declare this variable with an `env` in the dockerfile?