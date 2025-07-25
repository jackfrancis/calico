include ../../metadata.mk

PACKAGE_NAME ?= github.com/projectcalico/calico/third_party/envoy-ratelimit

ENVOY_RATELIMIT_IMAGE ?= envoy-ratelimit
BUILD_IMAGES ?= $(ENVOY_RATELIMIT_IMAGE)

# For updating this version please see
# https://github.com/tigera/operator/blob/master/docs/common_tasks.md#updating-the-bundled-version-of-envoy-gateway
ENVOY_RATELIMIT_VERSION=0141a24f

##############################################################################
# Include lib.Makefile before anything else
#   Additions to EXTRA_DOCKER_ARGS need to happen before the include since
#   that variable is evaluated when we declare DOCKER_RUN and siblings.
##############################################################################
include ../../lib.Makefile

##############################################################################
# Build
##############################################################################
ENVOY_RATELIMIT_DOWNLOADED=.envoy-ratelimit.downloaded

.PHONY: init-source
init-source:: $(ENVOY_RATELIMIT_DOWNLOADED)
$(ENVOY_RATELIMIT_DOWNLOADED):
	git clone -n https://github.com/envoyproxy/ratelimit.git envoy-ratelimit
	cd envoy-ratelimit && git checkout $(ENVOY_RATELIMIT_VERSION)
	patch -d envoy-ratelimit -p1 < patches/0001-Bump-golang.org-x-deps.patch
	touch $@

.PHONY: build
build: bin/envoy-ratelimit-$(ARCH)

bin/envoy-ratelimit-$(ARCH): init-source
	$(DOCKER_GO_BUILD) \
		sh -c '$(GIT_CONFIG_SSH) \
			CGO_ENABLED=0 go build -C envoy-ratelimit -buildvcs=false -o ../$@ -v -tags=$(TAGS) -ldflags="$(LD_FLAGS) -s -w" github.com/envoyproxy/ratelimit/src/service_cmd'

.PHONY: clean
clean:
	rm -f .envoy-ratelimit.* .*.created* .*.published* .release.*
	rm -fr bin/ envoy-ratelimit/
	-docker image rm -f $$(docker images $(ENVOY_RATELIMIT_IMAGE) -a -q)

##############################################################################
# Image
##############################################################################
ENVOY_RATELIMIT_IMAGE_CREATED=.envoy-ratelimit.created-$(ARCH)

.PHONY: image-all
image-all: $(addprefix sub-image-,$(VALIDARCHES))
sub-image-%:
	$(MAKE) image ARCH=$*

.PHONY: image
image: $(BUILD_IMAGES)

$(ENVOY_RATELIMIT_IMAGE): $(ENVOY_RATELIMIT_IMAGE_CREATED)
$(ENVOY_RATELIMIT_IMAGE_CREATED): Dockerfile build
	$(DOCKER_BUILD) -t $(ENVOY_RATELIMIT_IMAGE):latest-$(ARCH) -f Dockerfile .
	$(MAKE) retag-build-images-with-registries VALIDARCHES=$(ARCH) IMAGETAG=latest
	touch $@

##############################################################################
# CI/CD
##############################################################################
ci: image

cd: image-all cd-common

.PHONY: release-build
release-build: .release-$(VERSION).created
.release-$(VERSION).created:
	$(MAKE) clean image-all RELEASE=true
	$(MAKE) retag-build-images-with-registries IMAGETAG=$(VERSION) RELEASE=true
	# Generate the `latest` images.
	$(MAKE) retag-build-images-with-registries IMAGETAG=latest RELEASE=true
	touch $@

release-publish: release-prereqs .release-$(VERSION).published
.release-$(VERSION).published:
	$(MAKE) push-images-to-registries push-manifests IMAGETAG=$(VERSION) RELEASE=$(RELEASE) CONFIRM=$(CONFIRM)
	touch $@
