SUDO=sudo
GOFLAGS=-mod=vendor
export GOFLAGS
GINKGO=$(GOPATH)/bin/ginkgo
KUBECTL=/usr/local/bin/kubectl
KUBERNETES_VERSION=1.18.2
KUSTOMIZE_VERSION= 3.6.1
KIND_VERSION=0.8.1
KIND_CLUSTER_NAME=topolvm-example

GO_FILES := $(shell find .. -path ../vendor -prune -o -path ../e2e -prune -o -name '*.go' -print)
CA_FILES=./build/certs/ca.csr ./build/certs/ca.pem ./build/certs/ca-key.pem
SERVER_CERT_FILES=./build/certs/server.csr ./build/certs/server.pem ./build/certs/server-key.pem

$(KUBECTL):
	$(SUDO) curl -sfL https://storage.googleapis.com/kubernetes-release/release/v$(KUBERNETES_VERSION)/bin/linux/amd64/kubectl -o $(KUBECTL)
	$(SUDO) chmod 755 $(KUBECTL)

run: $(SERVER_CERT_FILES)
	$(MAKE) start-lvmd
	$(MAKE) launch-kind
	kustomize build . | $(KUBECTL) apply -f -
	$(KUBECTL) wait --for=condition=available --timeout=120s -n topolvm-system deployments/controller
	$(KUBECTL) apply -f podpvc.yaml

setup: $(KUBECTL)
	cd /tmp; env GOFLAGS= GO111MODULE=on go get sigs.k8s.io/kind@v$(KIND_VERSION)
	go install github.com/cloudflare/cfssl/cmd/cfssl
	go install github.com/cloudflare/cfssl/cmd/cfssljson
	$(SUDO) apt-get update
	$(SUDO) apt-get install -y lvm2 xfsprogs
	if apt-cache show btrfs-progs; then \
		$(SUDO) apt-get install -y btrfs-progs; \
	else \
		$(SUDO) apt-get install -y btrfs-tools; \
	fi
	curl -sSLf https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv$(KUSTOMIZE_VERSION)/kustomize_v$(KUSTOMIZE_VERSION)_linux_amd64.tar.gz | tar zxf - -C /tmp
	$(SUDO) mv /tmp/kustomize /usr/local/bin/kustomize
	chmod +x /usr/local/bin/kustomize

clean: stop-lvmd
	rm -rf build/

/tmp/topolvm/scheduler/scheduler-config.yaml: ../deploy/scheduler-config/scheduler-config.yaml
	mkdir -p /tmp/topolvm/scheduler
	cp $< $@

/tmp/topolvm/scheduler/scheduler-policy.cfg: ../deploy/scheduler-config/scheduler-policy.cfg
	mkdir -p /tmp/topolvm/scheduler
	cp $< $@

/tmp/topolvm/lvmd/lvmd.yaml: ../deploy/lvmd-config/lvmd.yaml
	mkdir -p /tmp/topolvm/lvmd
	sed -e 's/\/run\/topolvm\/lvmd.sock/\/tmp\/topolvm\/lvmd\/lvmd.sock/' $< > $@

launch-kind: /tmp/topolvm/scheduler/scheduler-config.yaml /tmp/topolvm/scheduler/scheduler-policy.cfg kind/topolvm-cluster.yaml
	$(MAKE) shutdown-kind
	$(SUDO) rm -rf /tmp/topolvm/controller /tmp/topolvm/worker*
	sed s/@KUBERNETES_VERSION@/$(KUBERNETES_VERSION)/ kind/topolvm-cluster.yaml > /tmp/topolvm-cluster.yaml
	kind create cluster --name=$(KIND_CLUSTER_NAME) --config /tmp/topolvm-cluster.yaml --image kindest/node:v$(KUBERNETES_VERSION)

shutdown-kind:
	kind delete cluster --name=$(KIND_CLUSTER_NAME) || true
	for d in $$($(SUDO) find /tmp/topolvm -type d); do \
		if $(SUDO) mountpoint -q $$d; then \
			$(SUDO) umount $$d; \
		fi; \
	done

start-lvmd: /tmp/topolvm/lvmd/lvmd.yaml
	go build -o build/lvmd ../pkg/lvmd
	if [ -f build/backing_store ]; then $(MAKE) stop-lvmd; fi
	mkdir -p /tmp/topolvm/worker; \
	mkdir -p /tmp/topolvm/lvmd; \
	truncate --size=20G build/backing_store; \
	$(SUDO) losetup -f build/backing_store; \
	$(SUDO) vgcreate -y myvg1 $$($(SUDO) losetup -j build/backing_store | cut -d: -f1); \
	$(SUDO) systemd-run --unit=lvmd.service $(shell pwd)/build/lvmd --config=/tmp/topolvm/lvmd/lvmd.yaml; \

stop-lvmd:
	$(MAKE) shutdown-kind
	if systemctl is-active -q lvmd.service; then $(SUDO) systemctl stop lvmd.service; fi; \
	if [ -f build/backing_store ]; then \
		$(SUDO) vgremove -ffy myvg1; \
		$(SUDO) pvremove -ffy $$($(SUDO) losetup -j build/backing_store | cut -d: -f1); \
		$(SUDO) losetup -d $$($(SUDO) losetup -j build/backing_store | cut -d: -f1); \
		rm -f build/backing_store; \
	fi; \

$(CA_FILES): ./certs/csr.json
	mkdir -p build/certs
	cfssl gencert -initca certs/csr.json | cfssljson -bare ./build/certs/ca

$(SERVER_CERT_FILES): $(CA_FILES) ./certs/ca-config.json ./certs/server.json
	cfssl gencert -ca=build/certs/ca.pem -ca-key=build/certs/ca-key.pem -config=certs/ca-config.json -profile=server certs/server.json | cfssljson -bare build/certs/server

.PHONY: launch-kind shutdown-kind start-lvmd stop-lvmd run setup clean
