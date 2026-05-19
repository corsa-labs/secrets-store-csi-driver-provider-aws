# .env File Mount Support

This fork extends the AWS Secrets Store CSI Driver Provider to write a `.env`
file alongside the individual secret files on every mount. The `.env` file
contains all mounted secrets as `KEY=VALUE` pairs, one per line.

## Why

The upstream driver writes one file per secret. Getting those secrets into
environment variables requires either:

- Kubernetes `Secret` objects synced via `secretObjects` (beta, AWS themselves
  advise against it)
- An init container that reads the files and writes a `.env` (fragile, hard to
  troubleshoot at 2am)

This fork eliminates both workarounds. Runtimes like [Bun](https://bun.sh)
load `.env` automatically. Node apps using `dotenv` pick it up with zero code
changes.

## What Changed

`server/server.go` — after writing individual secret files, `writeDotEnv` is
called to produce a single `.env` in the mount directory:

```
ANTHROPIC_API_KEY=sk-ant-...
AUTH0_CLIENT_SECRET=abc123...
```

## Usage

### SecretProviderClass

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: my-app-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "MY_API_KEY"
        objectType: "secretsmanager"
      - objectName: "IAC_AUTH0_CLIENT_SECRET"
        objectType: "secretsmanager"
        objectAlias: "AUTH0_CLIENT_SECRET"
```

### Deployment — mount `.env` directly into the app directory

Using `subPath`, only the `.env` file is projected into `/app`. The rest of
the container filesystem is untouched.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-app-sa
      volumes:
        - name: secrets-store
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: my-app-secrets
      containers:
        - name: my-app
          image: my-app:latest
          volumeMounts:
            - name: secrets-store
              mountPath: /app/.env
              subPath: .env
```

Bun picks up `/app/.env` automatically at startup. For Node + `dotenv`, no
config changes are needed as long as the app calls `dotenv.config()` (or uses
`dotenv/config`) before reading `process.env`.

## Notes

- Individual secret files are still written (backward compatible).
- `subPath` mounts do not live-update on secret rotation. Restart the pod to
  pick up new values.
- The `.env` file is written with `0644` permissions.
