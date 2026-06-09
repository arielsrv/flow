# go-app — example consumer of arielsrv/flow

A minimal Go HTTP service that demonstrates how to consume the shared CI/CD
pipeline from [arielsrv/flow](https://github.com/arielsrv/flow).

## How to use this example as a new repo

1. Create a new GitHub repository
2. Copy the contents of this folder to the repo root:
   ```
   main.go
   main_test.go
   go.mod
   .github/workflows/flow.yml
   ```
3. Push — the CI pipeline runs automatically via `arielsrv/flow`

## What the pipeline does

```
push / PR
    └─▶ arielsrv/flow / pipeline.yml
              └─▶ ci-go.yml
                    ├─ actions/setup-go  (version from go.mod)
                    ├─ go build ./...
                    └─ go test -v -cover ./...
```

## Local run

```bash
go run .
curl http://localhost:8080/health
# {"status":"healthy"}
```

## Local test

```bash
go test -v -cover ./...
```

