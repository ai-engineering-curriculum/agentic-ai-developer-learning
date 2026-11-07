# Lab 03 — Does `.env` Make It Into the Image? Proving `.dockerignore` Earns Its Line

**Estimated duration:** 30–45 minutes
**Covers:** [chapter 6 — Containerizing with Docker](../06-containerizing-with-docker.md) ("The `.dockerignore`" and "Where secrets go")

## Objective

Chapter 6 picks exactly one line as *"the single most important"* in the whole `.dockerignore`:

> `.env` is excluded. This is the single most important line. A `COPY . .` that includes `.env` bakes your API key into the image — and an image is meant to be shared. Excluding `.env` is non-negotiable.

Chapter 1 reinforces the same claim from a different angle: *"A Docker image built once and shared is a public-ish artifact even when it's private. Bake your key into the image and it's only one accidental `docker push` away from a public registry."*

In this lab you will build the *same* Dockerfile against the *same* source tree **twice**: once with `.dockerignore` excluding `.env`, and once without. Each build produces an image. You will then inspect both images the way a security reviewer would — poking at the layered filesystem, reading `docker history`, and running a container that just `cat`s whatever files are inside. One image contains the key. One does not. The lab's point is that *nothing about the Dockerfile itself changed* — the only difference was one text file next to it.

This lab complements [exercise-03](../exercises/exercise-03-logging-and-usage-guardrail.md), whose smoke test includes a step named "No secret in the image" and asks you to run `docker history | grep -i anthropic`. The exercise checks a pass. The lab makes the failure visible first, so the pass means something.

By the end you will have:

- A tiny repo (`dockerignore-lab/`) with a Dockerfile, a `.env` file containing a sentinel (fake) secret, and two `.dockerignore` files you swap between builds.
- Two local images (`leaky:0.1` and `clean:0.1`) built from the same source.
- A short `NOTES.md` recording: the sentinel visible in `leaky`, the sentinel absent from `clean`, the image-size delta (if any), and a one-paragraph answer about why `docker history` on its own is not a sufficient check.

## Prerequisites

- Finished chapter 6. You should recognize `COPY . .`, `.dockerignore`, `docker history`, and multi-stage builds on sight.
- Docker Desktop, Podman, or colima installed and able to `docker build` and `docker run` a Python image. Verify: `docker run --rm hello-world`.
- `grep`, `jq`, and `python3` on your PATH.
- Budget: $0. All work is local; no image is pushed anywhere.
- **No real API key.** This lab uses a **fake** sentinel string (`sk-ant-FAKE-SENTINEL-DO-NOT-USE`) so you can safely search for it later without worrying about leaking a real credential into your shell history or notes.

## Step 1 — Build the lab repo

Make a scratch directory and drop these four files:

```bash
mkdir dockerignore-lab && cd dockerignore-lab
```

`Dockerfile` — a minimal shape, close to chapter 6's multi-stage version but trimmed for lab speed:

```dockerfile
# Dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1

WORKDIR /app

# The star of the show — a broad COPY that will take whatever
# the build context gives it:
COPY . .

CMD ["python", "-c", "print('hello from clean image')"]
```

`app.py` — one line, just so the "source tree" is not empty:

```python
# app.py
print("app says hi")
```

`.env` — the sentinel secret. **Do not put a real key here.**

```bash
# .env   <-- NEVER commit a real file like this; this is a lab-only fake.
ANTHROPIC_API_KEY=sk-ant-FAKE-SENTINEL-DO-NOT-USE
DAILY_USD_CAP=5.0
LOG_LEVEL=INFO
```

`.dockerignore.clean` — the chapter 6 recommended shape:

```
# .dockerignore.clean  (the "clean" build will copy this to .dockerignore)
.env
.env.*
!.env.example

.git
__pycache__/
*.pyc
```

`.dockerignore.leaky` — the mistake. Deliberately empty so nothing is excluded:

```
# .dockerignore.leaky  (the "leaky" build will copy this to .dockerignore)
# (intentionally empty — this is the broken config under test)
```

### Checkpoint

- Four files in the directory: `Dockerfile`, `app.py`, `.env`, `.dockerignore.clean`, `.dockerignore.leaky`. (`.env.example` is optional for this lab.)
- `cat .env | grep ANTHROPIC_API_KEY` shows the sentinel. If you see a real-looking key here you accidentally put a real one in — rotate it, and replace with the sentinel before continuing.
- Nothing in this directory is a git repo. If it is, make sure `.env` is in `.gitignore` before any `git add`.

## Step 2 — Build the leaky image

Activate the broken config:

```bash
cp .dockerignore.leaky .dockerignore
docker build -t leaky:0.1 .
```

Watch the build output. The `COPY . .` step will mention how large the build context is — if you see a `Sending build context to Docker daemon` line, note the size (in `NOTES.md`).

Now the interesting part — run a container from the image and `cat` the file that should not be there:

```bash
docker run --rm leaky:0.1 cat /app/.env
# ANTHROPIC_API_KEY=sk-ant-FAKE-SENTINEL-DO-NOT-USE
# DAILY_USD_CAP=5.0
# LOG_LEVEL=INFO
```

That is the failure chapter 6 names. The sentinel is in the image's filesystem. It will be in the image anywhere the image goes — a `docker push`, a `docker save > image.tar` shared by email, a CI artifact uploaded to an S3 bucket.

Three more checks, each in `NOTES.md`:

1. **`grep` the image via a filesystem export.** Save the image to a tar, open it, and look for the sentinel:

   ```bash
   docker save leaky:0.1 -o leaky.tar
   mkdir -p leaky-fs && tar -xf leaky.tar -C leaky-fs
   # Extract each layer blob; sentinel will be in one of them.
   for layer in leaky-fs/blobs/sha256/*; do
       if file "$layer" | grep -q 'tar archive'; then
           tar -tf "$layer" 2>/dev/null | grep -q '\.env$' \
               && echo "layer $layer contains .env"
       fi
   done
   ```

   At least one layer should print. Older Docker image formats use `layer.tar` files instead of `blobs/sha256/`; adapt the glob if your extract looks different.

2. **`docker history` with `--no-trunc`.** This shows the commands that built each layer but *not* the contents of `COPY`s:

   ```bash
   docker history leaky:0.1 --no-trunc | grep -i 'COPY\|anthropic' || echo "no match"
   ```

   You will likely see the `COPY . .` line but **no mention of the sentinel string**. Record this in `NOTES.md`. Chapter 6 reminds you that *"build-arg secrets are not secrets"* — they *do* show up in `history`. A secret that got in via `COPY` instead of `ARG` is invisible to `history`. Pretending `history` is a complete audit tool is the kind of mistake the chapter is trying to inoculate you against.

3. **`docker inspect` on the config.** Confirm the key is not in the container's declared environment (it shouldn't be — nothing in the Dockerfile sets `ENV ANTHROPIC_API_KEY=...`). The leak is in the *filesystem*, not the config:

   ```bash
   docker inspect leaky:0.1 | jq '.[0].Config.Env'
   # ["PATH=...", "LANG=C.UTF-8", "PYTHON_VERSION=...", ...]
   ```

   The sentinel is absent from `.Config.Env`. That is exactly why this failure is hard to catch — `docker inspect` passes; the filesystem is where the problem lives.

### Checkpoint

- `docker run --rm leaky:0.1 cat /app/.env` prints the sentinel.
- At least one tar layer in `leaky.tar` contains `.env` (step 2.1).
- `docker history --no-trunc` does *not* print the sentinel (step 2.2).
- `docker inspect ... .Config.Env` does *not* contain the sentinel (step 2.3).
- All three observations are in `NOTES.md`. Together they document that the leak is not where most reviewers look first.

## Step 3 — Build the clean image

Swap in the chapter 6 config and rebuild:

```bash
cp .dockerignore.clean .dockerignore
docker build -t clean:0.1 .
```

Rerun each of the three checks against `clean:0.1`:

```bash
# 3.1 - the file should not exist
docker run --rm clean:0.1 cat /app/.env
# cat: /app/.env: No such file or directory

# 3.2 - the layer-extract trick
docker save clean:0.1 -o clean.tar
mkdir -p clean-fs && tar -xf clean.tar -C clean-fs
for layer in clean-fs/blobs/sha256/*; do
    if file "$layer" | grep -q 'tar archive'; then
        tar -tf "$layer" 2>/dev/null | grep -q '\.env$' \
            && echo "layer $layer contains .env"
    fi
done
# (no output — no layer contains .env)

# 3.3 - grep ALL the layers for the sentinel string
grep -ar 'FAKE-SENTINEL' clean-fs/ || echo "sentinel not present anywhere in clean image"
```

The third check is the one that matters most — it searches every byte of every layer for the sentinel string. On the clean image it should return `sentinel not present anywhere in clean image`. For comparison, rerun it against the leaky export:

```bash
grep -ar 'FAKE-SENTINEL' leaky-fs/
# (should print at least one match in a layer tar)
```

Record both outputs in `NOTES.md` and a one-sentence note: *"grep across layer contents is the honest check; `docker history` and `docker inspect` miss this leak."*

### Checkpoint

- `docker run --rm clean:0.1 cat /app/.env` errors with "No such file or directory."
- `grep -ar FAKE-SENTINEL clean-fs/` prints no matches.
- `grep -ar FAKE-SENTINEL leaky-fs/` prints at least one match.
- The clean image size is smaller or similar to the leaky one. Record the sizes:
  ```bash
  docker images leaky:0.1 clean:0.1 --format '{{.Repository}}:{{.Tag}} {{.Size}}'
  ```
  The delta is tiny (a few hundred bytes), which is part of chapter 6's point — *this is not a size regression you would catch in a dashboard*.

## Step 4 — Inject the key the right way

Chapter 6 says the running container should have the secret *injected at runtime* via `--env-file` (or `--env`), never baked. Prove the pattern works on the clean image:

```bash
docker run --rm --env-file .env clean:0.1 \
    python -c 'import os; print("got key:", bool(os.environ.get("ANTHROPIC_API_KEY")))'
# got key: True

docker run --rm clean:0.1 \
    python -c 'import os; print("got key:", bool(os.environ.get("ANTHROPIC_API_KEY")))'
# got key: False
```

Two conclusions, both in `NOTES.md`:

- Without `--env-file`, the clean image has no access to `ANTHROPIC_API_KEY`. The key is *not* in the image.
- With `--env-file`, the runtime environment has it — exactly the shape chapter 3 and chapter 6 described: *"every layer presents the secret as an environment variable to the running process."*

Chapter 3's twelve-factor rule lands here: the same image runs anywhere; only the env differs.

### Checkpoint

- Both commands ran; one returned `True`, the other `False`.
- `NOTES.md` has the two conclusions and a brief note about the twelve-factor claim.

## Step 5 — What about `.env.example`?

Chapter 6's `.dockerignore` snippet has one more deliberate line: `!.env.example`. The `!` un-excludes the example so the image carries the self-describing contract. Reproduce it:

```bash
cat > .env.example <<'EOF'
# Required
ANTHROPIC_API_KEY=replace-me

# Optional
# DAILY_USD_CAP=5.0
# LOG_LEVEL=INFO
EOF

docker build -t clean:0.2 .
docker run --rm clean:0.2 ls -la /app
# -rw-r--r-- ... .env.example
# -rw-r--r-- ... app.py
```

Confirm:
- `.env.example` is in the image.
- `.env` is still absent.

Record in `NOTES.md`: *"The negation pattern in `.dockerignore` keeps the committed example in the image and the real secret out."*

### Checkpoint

- Both files visible / absent as expected.
- `NOTES.md` notes the negation-pattern mechanism.

## Step 6 — Clean up

You created a fake secret and two images. Nothing here was real, but clean up anyway so you don't accidentally `docker push` a lab image later:

```bash
docker rmi leaky:0.1 clean:0.1 clean:0.2
rm -rf leaky-fs clean-fs leaky.tar clean.tar
```

Leave the repo itself in place if you want to refer back to it.

### Checkpoint

- `docker images | grep -E 'leaky|clean'` is empty.
- `ls` in your lab directory does not show the extracted `leaky-fs/`, `clean-fs/`, or `*.tar` files.
- In `NOTES.md`, note what you cleaned up and what you kept.

## Debrief

Three things to take with you:

- **The leak is invisible to the obvious checks.** `docker history` does not show the key. `docker inspect` does not show the key. The honest check is `grep -r` across layer contents or `docker run <image> cat <path>`. Chapter 6's acceptance-criteria `docker history | grep -i anthropic` is a *necessary* smoke test, not a sufficient one.
- **One text file guards the whole image.** The Dockerfile did not change between `leaky:0.1` and `clean:0.1`. The source tree did not change. The *only* difference was a `.dockerignore` line, and that line is the difference between a shareable artifact and an accidental credential disclosure.
- **The image is a public-ish artifact.** Chapter 1 phrases it as *"a Docker image built once and shared is a public-ish artifact even when it's private."* Chapter 6 phrases it as *"only one accidental `docker push` away from a public registry."* The lab's sentinel is fake, but every subtlety of the disclosure (invisible to `history`, invisible to `inspect`, present in the filesystem of every pull) is real.

## Stretch

- **Add the image to a local registry and inspect the push.** `docker run -d -p 5000:5000 --name reg registry:2` starts a local registry; `docker tag leaky:0.1 localhost:5000/leaky:0.1 && docker push localhost:5000/leaky:0.1` pushes it. The push output names the layer hashes. Inside each layer, the sentinel is still there. The point is to feel what "pushed" means — the image is now addressable by anyone who can reach the registry.

- **Try the `ARG`-as-secret anti-pattern.** Add `ARG ANTHROPIC_API_KEY` and `ENV ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY` to the Dockerfile, build with `docker build --build-arg ANTHROPIC_API_KEY=sk-ant-ARG-SENTINEL -t argleak:0.1 .`, and then run `docker history argleak:0.1 --no-trunc | grep ARG-SENTINEL`. This time the key *is* visible in `history` (chapter 6 warns about this exactly). Note the two leaks in `NOTES.md` and which inspection tool catches which — `history` catches `ARG`-leaks; file-system grep catches `COPY`-leaks. One tool is not enough.

- **Rebuild as multi-stage.** Rewrite the Dockerfile in the chapter-6 two-stage form (`FROM python:3.12-slim AS build` → `FROM python:3.12-slim AS runtime` with a `COPY --from=build`). Keep `.dockerignore.leaky` active. Does the leak still happen? (Hint: the `COPY . .` is in the stage that ends up as the runtime, so yes.) Which `COPY` line in your new Dockerfile is the one you must audit? Chapter 6's claim is that multi-stage *does not* automatically protect against this — the hygiene is in `.dockerignore`, not the stage boundary.
