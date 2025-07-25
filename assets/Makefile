LOCAL_BIN:=$(CURDIR)/bin
APP_NAME:=sns_sim

.PHONY: run build build-run test install-golangci-lint lint validate-lint-config go-lic-install third_party_licenses install-godoc docker-up docker-build docker-rebuild

# run app
run:
	./bin/$(APP_NAME)

# build
build:
	go build -o ./bin/$(APP_NAME) ./cmd

# build and run
build-run:
	go build -o ./bin/$(APP_NAME) ./cmd && ./bin/$(APP_NAME)

# run tests
test:
	go test ./... -v

# install linter
install-golangci-lint:
	GOBIN=$(LOCAL_BIN) go install github.com/golangci/golangci-lint/cmd/golangci-lint@v1.64.8

# run linting
lint:
	$(LOCAL_BIN)/golangci-lint run --config=.golangci.yaml ./...

# validate golangci-lint config
validate-lint-config:
	$(LOCAL_BIN)/golangci-lint config verify --config=.golangci.yaml

# install go-licenses
go-lic-install:
	GOBIN=$(LOCAL_BIN) go install github.com/google/go-licenses@v1.6.0

# save 3rd party packages licenses to file
third_party_licenses:
	$(LOCAL_BIN)/go-licenses report ./... > THIRD_PARTY_LICENSES

# install godoc
install-godoc:
	GOBIN=$(LOCAL_BIN) go install golang.org/x/tools/cmd/godoc@v0.34.0

# Start the Docker containers in background
docker-up:
	docker-compose up -d

# Build images (if needed) and start the Docker containers in detached mode
docker-build:
	docker-compose up --build -d

# Stop and remove all containers and their associated volumes, then rebuild and start containers
docker-rebuild:
	docker-compose down --volumes
	make docker-build

