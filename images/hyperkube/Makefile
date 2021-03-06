REGISTRY?=luxas
IMAGE?=hyperkube
VERSION?=v1.6.0-and-PR-42911
COMMIT=b705835bae42925e0f9ed2a1fcf22d1d6b1dc680
ARCH?=amd64
ALL_ARCH=amd64 arm arm64
ML_PLATFORMS=linux/amd64,linux/arm,linux/arm64
TEMP_DIR:=$(shell mktemp -d)
QEMUVERSION=v2.7.0

ifeq ($(ARCH),arm)
	QEMUARCH=arm
endif
ifeq ($(ARCH),arm64)
	QEMUARCH=aarch64
endif

BASEIMAGE?=gcr.io/google-containers/debian-iptables-$(ARCH):v7

./kubernetes:
	git clone https://github.com/kubernetes/kubernetes

$(TEMP_DIR)/hyperkube: ./kubernetes
	cd kubernetes \
		&& git checkout $(COMMIT) \
		&& build/run.sh /bin/bash -c "KUBE_BUILD_PLATFORMS=linux/$(ARCH) make WHAT=cmd/hyperkube" \
		&& cp _output/dockerized/bin/linux/$(ARCH)/hyperkube $(TEMP_DIR)

build: $(TEMP_DIR)/hyperkube
	cp Dockerfile $(TEMP_DIR)
	cd $(TEMP_DIR) && sed -i "s|BASEIMAGE|$(BASEIMAGE)|g" Dockerfile
	cd $(TEMP_DIR) && sed -i "s|QEMUARCH|$(QEMUARCH)|g" Dockerfile

ifeq ($(ARCH),amd64)
	# When building "normally" for amd64, remove the whole line, it has no part in the amd64 image
	cd $(TEMP_DIR) && sed -i "/CROSS_BUILD_/d" Dockerfile
else
	# When cross-building, only the placeholder "CROSS_BUILD_" should be removed
	# Register /usr/bin/qemu-ARCH-static as the handler for ARM binaries in the kernel
	docker run --rm --privileged multiarch/qemu-user-static:register --reset
	curl -sSL https://github.com/multiarch/qemu-user-static/releases/download/$(QEMUVERSION)/x86_64_qemu-$(QEMUARCH)-static.tar.gz | tar -xz -C $(TEMP_DIR)
	cd $(TEMP_DIR) && sed -i "s/CROSS_BUILD_//g" Dockerfile
endif

	docker build --pull -t $(REGISTRY)/$(IMAGE)-$(ARCH):$(VERSION) $(TEMP_DIR)

push-%:
	$(MAKE) ARCH=$* build
	docker push $(REGISTRY)/$(IMAGE)-$*:$(VERSION)

push: ./manifest-tool $(addprefix push-,$(ALL_ARCH))
	./manifest-tool push from-args --platforms $(ML_PLATFORMS) --template $(REGISTRY)/$(IMAGE)-ARCH:$(VERSION) --target $(REGISTRY)/$(IMAGE):$(VERSION)

./manifest-tool:
	curl -sSL https://github.com/estesp/manifest-tool/releases/download/v0.4.0/manifest-tool-linux-amd64 > manifest-tool
	chmod +x manifest-tool
