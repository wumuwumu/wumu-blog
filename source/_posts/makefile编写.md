---
title: makefile编写
tags:
  - go
abbrlink: 3e37945d
date: 2019-04-10 10:27:53
---

# 例子

```
.PHONY: build clean test package package-deb ui api statics requirements ui-requirements serve update-vendor internal/statics internal/migrations static/swagger/api.swagger.json
PKGS := $(shell go list ./... | grep -v /vendor |grep -v lora-app-server/api | grep -v /migrations | grep -v /static | grep -v /ui)
VERSION := $(shell git describe --always |sed -e "s/^v//")

build: ui/build internal/statics internal/migrations
	mkdir -p build
	go build $(GO_EXTRA_BUILD_ARGS) -ldflags "-s -w -X main.version=$(VERSION)" -o build/lora-app-server cmd/lora-app-server/main.go

clean:
	@echo "Cleaning up workspace"
	@rm -rf build dist internal/migrations internal/static ui/build static/static
	@rm -f static/index.html static/icon.png static/manifest.json static/asset-manifest.json static/service-worker.js
	@rm -rf static/logo
	@rm -rf docs/public
	@rm -rf dist

test: internal/statics internal/migrations
	@echo "Running tests"
	@for pkg in $(PKGS) ; do \
		golint $$pkg ; \
	done
	@go vet $(PKGS)
	@go test -p 1 -v $(PKGS)

documentation:
	@echo "Building documentation"
	@mkdir -p dist/docs
	@cd docs && hugo
	@cd docs/public/ && tar -pczf ../../dist/lora-app-server-documentation.tar.gz .

dist: ui/build internal/statics internal/migrations
	@goreleaser

build-snapshot: ui/build internal/statics internal/migrations
	@goreleaser --snapshot

package-deb: package
	@echo "Building deb package"
	@cd packaging && TARGET=deb ./package.sh

ui/build:
	@echo "Building ui"
	@cd ui && npm run build
	@mv ui/build/* static

api:
	@echo "Generating API code from .proto files"
	@go generate api/api.go

internal/statics internal/migrations: static/swagger/api.swagger.json
	@echo "Generating static files"
	@go generate cmd/lora-app-server/main.go


static/swagger/api.swagger.json:
	@echo "Generating combined Swagger JSON"
	@GOOS="" GOARCH="" go run api/swagger/main.go api/swagger > static/swagger/api.swagger.json
	@cp api/swagger/*.json static/swagger


# shortcuts for development

requirements:
	echo "Installing development tools"
	go get -u github.com/golang/lint/golint
	go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-grpc-gateway
	go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-swagger
	go get -u github.com/golang/protobuf/protoc-gen-go
	go get -u github.com/elazarl/go-bindata-assetfs/...
	go get -u github.com/jteeuwen/go-bindata/...
	go get -u github.com/kisielk/errcheck
	go get -u github.com/smartystreets/goconvey
	go get -u golang.org/x/tools/cmd/stringer
	go get -u github.com/golang/dep/cmd/dep
	go get -u github.com/goreleaser/goreleaser
	dep ensure -v

ui-requirements:
	@echo "Installing UI requirements"
	@cd ui && npm install

serve: build
	@echo "Starting Lora App Server"
	./build/lora-app-server

update-vendor:
	@echo "Updating vendored packages"
	@govendor update +external

run-compose-test:
	docker-compose run --rm appserver make test
```

# 文件格式

```
<target> : <prerequisites> 
[tab]  <commands>
```

- target：执行的命令或者文件名。如果只是执行的命令这是`伪指令`，在大部分时候使用`.PHONY`声明伪指令，这样不仅仅提供效率，同时也避免和文件名冲突。
- prerequisites：前置条件。
- commands：需要执行的命令，
  - 前面需要添加`[tab]`，如果想要换成其他的，使用`.RECIPEPREFIX = ？`换成你喜欢的。
  - 执行命令的时候会打印出相关的命令内容，这个叫做`回显`，如果不想显示出来可以在命令前面添加`@`。
  - 命令执行的时候，每行命令在不同一个shell中执行，如果想在同一个shell中执行，有下面几个办法。
  - 将命令写在同一行
  - 在命令后面添加`\`，实现命令多行
  - 使用`.ONESHELL:`

# 内置变量

makefile可以通过`=、:=、?=、+=`给变量赋值，同时Make命令提供一系列内置变量，比如，\((CC)指向当前使用的编译器，\)(MAKE) 指向当前使用的Make工具。这主要是为了跨平台的兼容性，详细的内置变量清单见[手册](https://www.gnu.org/software/make/manual/html_node/Implicit-Variables.html)。

# 参考

https://blog.csdn.net/u010230971/article/details/80335613

https://www.cnblogs.com/wang_yb/p/3990952.html

http://www.ruanyifeng.com/blog/2015/02/make.html

