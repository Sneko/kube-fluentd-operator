# Copyright © 2018 VMware, Inc. All Rights Reserved.
# SPDX-License-Identifier: BSD-2-Clause

IMAGE          ?= vmware/kube-fluentd-operator
TAG            ?= latest

VERSION       ?= $(shell git describe --tags --always --dirty)
BUILD_FLAGS   := -v
LDFLAGS       := -X github.com/vmware/kube-fluentd-operator/config-reloader/config.Version=$(VERSION) -w -s

-include vmware.mk
.DEFAULT_GOAL := build-image

install:
	go install -v .

build:
	CGO_ENABLED=0 go build $(BUILD_FLAGS) -ldflags "$(LDFLAGS)" .

test:
	go test -v ./...

dep:
	which dep > /dev/null || (echo "Install dep first: go get -u github.com/golang/dep/cmd/dep" && exit 1)
	dep ensure

guess-tag:
	@echo "TAG=`git describe --tags --always`-`date -u +"%Y%m%d"`"

clean:
	rm -fr config-reloader pkg > /dev/null

build-image:
	cat Dockerfile | \
	  VERSION=$(VERSION) envsubst '$$VERSION' | \
	  docker build . -f - -t $(IMAGE):$(TAG)

push-image: build-image
	docker push $(IMAGE):$(TAG)


create-test-ns:
	HUMIO_KEY=$(HUMIO_KEY) LOGGLY_TOKEN=$(LOGGLY_TOKEN) envsubst '$$LOGGLY_TOKEN::$$HUMIO_KEY' < examples/manifests/kfo-test.yaml | kubectl apply -f -

delete-test-ns:
	kubectl delete -f examples/manifests/kfo-test.yaml

run-loop-fs: install
	rm -fr tmp
	config-reloader \
	  --interval 5 \
	  --log-level=debug \
	  --output-dir=tmp \
	  --meta-key=prefix \
	  --meta-values=aws_region=us-west-2,csp_cluster=mon \
	  --templates-dir=templates \
	  --datasource=fs \
	  --fs-dir=examples \
	  --fluentd-binary "fluentd/fake-fluentd.sh -p /plugins"

run-once-fs: install
	rm -fr tmp
	config-reloader \
	  --interval 0 \
	  --log-level=debug \
	  --output-dir=tmp \
	  --meta-key=prefix2 \
	  --meta-values=aws_region=us-west-2,csp_cluster=mon \
	  --templates-dir=templates \
	  --datasource=fs \
	  --fs-dir=examples \
	  --fluentd-binary "fluentd/fake-fluentd.sh -p /plugins"

run-once: install
	rm -fr tmp
	config-reloader \
	  --interval 0 \
	  --log-level=debug \
	  --output-dir=tmp \
	  --templates-dir=templates \
	  --meta-key=run-once \
	  --meta-values=aws_region=us-west-2,csp_cluster=mon \
	  --fluentd-binary "fluentd/fake-fluentd.sh -p /plugins"

run-loop: install
	rm -fr tmp
	config-reloader \
	  --interval 5 \
	  --log-level=debug \
	  --output-dir=tmp \
	  --meta-key=prefix3 \
	  --meta-values=aws_region=us-west-2,csp_cluster=mon \
	  --templates-dir=templates

run-fluentd:
	docker run --entrypoint=fluentd \
	  -ti --rm -v `pwd`:/workspace --net=host \
	  $(IMAGE):$(TAG) \
	  -p /fluentd/plugins -v -c /workspace/local-fluent.conf

validate-config:
	docker run --entrypoint=fluentd \
	  -ti --rm -v `pwd`:/workspace --net=host \
	  $(IMAGE):$(TAG) \
	  --dry-run -p /fluentd/plugins -v -c /workspace/tmp/fluent.conf

shell:
	docker run --entrypoint=/bin/bash \
	  -ti --rm -v `pwd`:/workspace --net=host \
	  $(IMAGE):$(TAG)
