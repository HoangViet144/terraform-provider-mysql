TEST?=$$(go list ./... |grep -v 'vendor')
GOFMT_FILES?=$$(find . -name '*.go' |grep -v vendor)
WEBSITE_REPO=github.com/hashicorp/terraform-website
PKG_NAME=mysql
TERRAFORM_VERSION=0.14.7
TEST_PASSWORD=my-secret-pw

default: build

build: fmtcheck
	go install

test: fmtcheck
	go test -i $(TEST) || exit 1
	echo $(TEST) | \
		xargs -t -n4 go test $(TESTARGS) -timeout=30s -parallel=4

bin/terraform:
	mkdir "$(CURDIR)/bin"
	curl https://releases.hashicorp.com/terraform/$(TERRAFORM_VERSION)/terraform_$(TERRAFORM_VERSION)_linux_amd64.zip > $(CURDIR)/bin/terraform.zip
	(cd $(CURDIR)/bin/ ; unzip terraform.zip)

testacc: fmtcheck bin/terraform
	PATH="$(CURDIR)/bin:${PATH}" TF_ACC=1 go test $(TEST) -v $(TESTARGS) -timeout=30s

acceptance: testversion5.6 testversion5.7 testversion8.0

testversion%:
	$(MAKE) MYSQL_VERSION=$* MYSQL_PORT=33$(shell echo "$*" | tr -d '.') testversion

testversion:
	-docker run --rm --name test-mysql$(MYSQL_VERSION) -e MYSQL_ROOT_PASSWORD="$(TEST_PASSWORD)" -d -p $(MYSQL_PORT):3306 mysql:$(MYSQL_VERSION)
	@while ! mysql -h 127.0.0.1 -P $(MYSQL_PORT) -p"$(TEST_PASSWORD)" -u root -e 'SELECT 1'; do echo 'Waiting for MySQL...'; sleep 1; done
	-mysql -h 127.0.0.1 -u root -P $(MYSQL_PORT) -p"$(TEST_PASSWORD)" -e "INSTALL PLUGIN mysql_no_login SONAME 'mysql_no_login.so';"
	MYSQL_PASSWORD="$(TEST_PASSWORD)" MYSQL_ENDPOINT=127.0.0.1:$(MYSQL_PORT) $(MAKE) testacc
	docker rm -f test-mysql$(MYSQL_VERSION)

vet:
	@echo "go vet ."
	@go vet $$(go list ./... | grep -v vendor/) ; if [ $$? -eq 1 ]; then \
		echo ""; \
		echo "Vet found suspicious constructs. Please check the reported constructs"; \
		echo "and fix them if necessary before submitting the code for review."; \
		exit 1; \
	fi

fmt:
	gofmt -w $(GOFMT_FILES)

deps:
	go mod tidy
	go mod vendor

fmtcheck:
	@sh -c "'$(CURDIR)/scripts/gofmtcheck.sh'"

errcheck:
	@sh -c "'$(CURDIR)/scripts/errcheck.sh'"

vendor-status:
	@govendor status

test-compile:
	@if [ "$(TEST)" = "./..." ]; then \
		echo "ERROR: Set TEST to a specific package. For example,"; \
		echo "  make test-compile TEST=./$(PKG_NAME)"; \
		exit 1; \
	fi
	go test -c $(TEST) $(TESTARGS)

website:
ifeq (,$(wildcard $(GOPATH)/src/$(WEBSITE_REPO)))
	echo "$(WEBSITE_REPO) not found in your GOPATH (necessary for layouts and assets), get-ting..."
	git clone https://$(WEBSITE_REPO) $(GOPATH)/src/$(WEBSITE_REPO)
endif
	@$(MAKE) -C $(GOPATH)/src/$(WEBSITE_REPO) website-provider PROVIDER_PATH=$(shell pwd) PROVIDER_NAME=$(PKG_NAME)

.PHONY: build test testacc vet fmt fmtcheck errcheck vendor-status test-compile website website-test

