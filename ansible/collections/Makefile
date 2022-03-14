# Based on that awesome makefile https://github.com/dunglas/symfony-docker/blob/main/docs/makefile.md#the-template

# Let user override python binary if needed
PYTHON?=python
# Ability to enable coverage where it's doable
WITH_COVERAGE?=0
# Enable docker for ansible tests
WITH_DOCKER?=0
# Python target for ansible tests
WITH_PY_TARGET?=
# Coverage report type (default to console report)
WITH_COVERAGE_REPORT?=console

COLLECTION_NAMESPACE=$(shell $(PYTHON) -c 'import yaml;print(yaml.load(open("galaxy.yml"), yaml.SafeLoader)["namespace"])')
COLLECTION_NAME=$(shell $(PYTHON) -c 'import yaml;print(yaml.load(open("galaxy.yml"), yaml.SafeLoader)["name"])')
COLLECTION_VERSION=$(shell $(PYTHON) -c 'import yaml;print(yaml.load(open("galaxy.yml"), yaml.SafeLoader)["version"])')

ANSIBLE_VERSION=$(shell $(PYTHON) -c 'from ansible import __version__ ;print("%s" % ".".join( __version__.split(".")[:2]));')
# ansible-core below v2.11 doesn't support installing fom current directory if it is not under C_NAMESPACE/C_NAME directories
# Install from tar.gz instead
ifeq ($(ANSIBLE_VERSION), 2.9)
ANSIBLE_INSTALL_OLD_FASHION=1
else ifeq ($(ANSIBLE_VERSION), 2.10)
ANSIBLE_INSTALL_OLD_FASHION=1
else
ANSIBLE_INSTALL_OLD_FASHION=0
endif

BUILD_DIR?=build
ANSIBLE_COLLECTIONS_BUILD_DIR=${BUILD_DIR}/ansible_collections
COLLECTION_BUILD_DIR=${ANSIBLE_COLLECTIONS_BUILD_DIR}/${COLLECTION_NAMESPACE}/${COLLECTION_NAME}

PYTHON_PKG_BUILD_DIR=/tmp/python-pkg/${COLLECTION_NAMESPACE}-${COLLECTION_NAME}
PYTHON_PKG_NAME=local-${COLLECTION_NAMESPACE}-${COLLECTION_NAME}-ansible-collections

ifeq ($(WITH_COVERAGE),1)
ANSIBLE_COVERAGE_OPTION=--coverage
else
ANSIBLE_COVERAGE_OPTION=
endif
ifeq ($(WITH_DOCKER),1)
ANSIBLE_DOCKER_OPTION=--docker
else
ANSIBLE_DOCKER_OPTION=
endif
ifneq ($(WITH_PY_TARGET),)
ANSIBLE_PY_TARGET_OPTION=--python $(WITH_PY_TARGET)
else
ANSIBLE_PY_TARGET_OPTION=
endif

.DEFAULT_GOAL = default

.PHONY: default
default: clean configure-dev-env

##—— 📚 Help ——————————————————————————————————————————————————————————————
.PHONY: help
help: infos ## ❓ Display this help
	@grep -E '(^[a-zA-Z0-9_-]+:.*?##.*$$)|(^##)' $(MAKEFILE_LIST) \
		| awk 'BEGIN {FS = ":.*?## "}{printf "\033[32m%-30s\033[0m %s\n", $$1, $$2}' \
		| sed -e 's/\[32m##——————————/[33m           /'  \
		| sed -e 's/\[32m##——/[33m ——/' \
		| sed -e 's/\[32m####/[34m                                 /' \
		| sed -e 's/\[32m###/[36m                                 /' \
		| sed -e 's/\[32m##\?/[35m /'  \
		| sed -e 's/\[32m##/[33m/'
infos: ## ℹ️  Display configuration
	@echo "~~~ Ansible collection ${COLLECTION_NAMESPACE}.${COLLECTION_NAME} (v${COLLECTION_VERSION}) ~~~"
	@echo "    ℹ️  Python binary: $(PYTHON)"
	@echo "    ℹ️  Ansible: $(ANSIBLE_VERSION)"
	@echo "    ℹ️  Build directory: $(BUILD_DIR)"
	@echo ""

##—— ️⚙️  Environments ——————————————————————————————————————————————————————
.PHONY: clean
clean: ## 🧹 Clean generated contents
clean:
	rm -rf build/* tests/output .mypy_cache ${COLLECTION_NAMESPACE}-${COLLECTION_NAME}-*.tar.gz

.PHONY: configure-dev-env
configure-dev-env: ## 🤖 Install required libraries for dev environment (python libs used on codebase)
configure-dev-env:
	$(PYTHON) -m pip install --upgrade --upgrade-strategy eager -r meta/ee-requirements.txt # Install internal requirements

.PHONY: configure-test-env
configure-test-env: ## 🤖 Install required libraries for test environment (libs used on codebase, MyPy, etc)
#### Use target="..." to restrict extra galaxy requirements installation to a specific target (units, integration, sanity)
$(eval target ?=)
configure-test-env: configure-dev-env
	$(PYTHON) -m pip install --upgrade --upgrade-strategy eager -r tests/requirements.txt # Install tests requirements
ifeq ($(shell if ([ "$(target)" = "units" ] || [ "$(target)" = "" ]) && [ -f tests/unit/requirements.yml ]; then echo 1; else echo 0; fi),1)
	$(MAKE) install install_o="-p ${ANSIBLE_COLLECTIONS_BUILD_DIR} -r tests/unit/requirements.yml" upgrade=1 source="--"
endif
ifeq ($(shell if ([ "$(target)" = "sanity" ] || [ "$(target)" = "" ]) && [ -f tests/sanity/requirements.yml ]; then echo 1; else echo 0; fi),1)
	$(MAKE) install install_o="-p ${ANSIBLE_COLLECTIONS_BUILD_DIR} -r tests/sanity/requirements.yml" upgrade=1 source="--"
endif
ifeq ($(shell if ([ "$(target)" = "integration" ] || [ "$(target)" = "" ]) && [ -f tests/integration/requirements.yml ]; then echo 1; else echo 0; fi),1)
	$(MAKE) install install_o="-p ${ANSIBLE_COLLECTIONS_BUILD_DIR} -r tests/integration/requirements.yml" upgrade=1 source="--"
endif

##—— 🇦 Ansible collection —————————————————————————————————————————————————
##—————————— \_ 🛰️  Galaxy —————————————————————————————————————————————————
.PHONY: build
build: ## 🗜️  Build for galaxy deployment (creates a tar.gz file)
#### Use build_o="..." to specify build options (--force, --output-path PATH, --token TOKEN, etc)
$(eval build_o ?=)
build: clean
	ansible-galaxy collection build $(build_o) .

.PHONY: install
install: ## ✍️  Install to a directory (ansible collection directory by default)
### Install from sources for Ansible 2.11+. Fallback on install from tar.gz for Ansible 2.10 and below
#### Use install_o="..." to specify install options (--force, -p PATH, --no-deps, etc)
$(eval install_o ?=)
#### Use upgrade=1 instead of install_o="--upgrade" to keep compatibility with ansible below 2.10
$(eval upgrade ?= 0)
#### Use source="..." to specify the source (tar.gz path mostly, default to current directory)
$(eval source ?= .)
ifeq ($(source),.)
# Clean project in case goal is to install current project
install: clean
endif
install:
ifeq ($(shell if [ "$(ANSIBLE_INSTALL_OLD_FASHION)" = "1" ] && [ "$(upgrade)" = "1" ]; then echo 1; else echo 0; fi),1)
	@echo "###################################################################################################"
	@echo "# --upgrade asked, not compatible with old fashion install. Existing content will be kept as is ! #"
	@echo "###################################################################################################"
else ifeq ($(upgrade),1)
	UPGRADE_OPT=" --upgrade"
endif
ifeq ($(shell if [ "$(ANSIBLE_INSTALL_OLD_FASHION)" = "1" ] && [ "$(source)" = "." ]; then echo 1; else echo 0; fi),1)
	@echo "#########################################"
	@echo "# Old fashion installation using tar.gz #"
	@echo "#########################################"
	$(MAKE) build build_o="--output-path ${BUILD_DIR} --force"
	ansible-galaxy collection install $(install_o) ${BUILD_DIR}/${COLLECTION_NAMESPACE}-${COLLECTION_NAME}-${COLLECTION_VERSION}.tar.gz
else
	ansible-galaxy collection install $(install_o)$$UPGRADE_OPT $(source)
endif

.PHONY: deploy
deploy: ## 🚀 Deploy to ansible galaxy
	@echo "TODO"

##—————————— \_ 🧪️ Tests ——————————————————————————————————————————————————
.PHONY: build-for-test
build-for-test: ## 🧪️ Build to the temporary build directory for test usage
$(eval install_o ?=)
build-for-test:
	rm -rf ${COLLECTION_BUILD_DIR} # Remove only the collection directory (dependencies will be kept there if previously installed)
	$(MAKE) install install_o="--force -p ${ANSIBLE_COLLECTIONS_BUILD_DIR}" upgrade=1 # Use upgrade=1 to always run on latest versions
	cd ${COLLECTION_BUILD_DIR} && git init -q . # Workaround when test folder is under a gitignored folder (else ansible-test does nothing)

##—————————— \_ 🐍 Python —————————————————————————————————————————————————
.PHONY: install-as-python-pkg
install-as-python-pkg: ## 🔗 Build the collection and mount content as python package to current python environment
install-as-python-pkg: clean
	rm -rf $(PYTHON_PKG_BUILD_DIR)/*
	$(MAKE) install install_o="--no-deps --force -p $(PYTHON_PKG_BUILD_DIR)/$(PYTHON_PKG_NAME)"
	cd $(PYTHON_PKG_BUILD_DIR) && $(PYTHON) -c 'from setuptools import setup;setup(name="$(PYTHON_PKG_NAME)", version="${COLLECTION_VERSION}", package_dir={"": "$(PYTHON_PKG_NAME)"}, packages=["ansible_collections"])' develop
	@echo ""
	@echo ""
	@echo "~~ Successfully installed $(PYTHON_PKG_NAME) (v$(COLLECTION_VERSION)) ~~"
	@echo ""

.PHONY: uninstall-python-pkg
uninstall-python-pkg: ## 🗑️  Uninstall python package mirroring the project
uninstall-python-pkg:
	$(PYTHON)  -m pip uninstall local-${COLLECTION_NAMESPACE}-${COLLECTION_NAME}-ansible-collections
	$(MAKE) clean


##—— 🧪️ Tests —————————————————————————————————————————————————————————————
.PHONY: test
test: ## 🏃 Launch all tests
test: test-python test-ansible

##—————————— \_ 🐍 Python —————————————————————————————————————————————————
.PHONY: test-python
test-python: ## 🏃 Launch python-related tests (typehint, lint, etc)
test-python: test-mypy

.PHONY: test-mypy
test-mypy: ## 🏃 Launch MyPy checks
test-mypy:
	$(PYTHON) -m mypy --config-file mypy.ini .
	$(PYTHON) -m mypy --config-file tests/mypy.ini tests

##—————————— \_ 🇦 Ansible tests ———————————————————————————————————————————
### Trigger ansible-related tests upon built collection
### Don't forget to execute build-for-test target before !
.PHONY: test-ansible
test-ansible: ## 🏃 Launch ansible-related tests (unit, integration and sanity tests)
### Unit and integration tests will be triggered only if related test directory exist!
test-ansible: build-for-test
test-ansible:
ifneq ($(wildcard tests/unit),) # Execute tests only if there is the required directory !
	$(MAKE) test-ansible-units
else
	@echo '################################################################'
	@echo '# Unit test deactivated as "tests/unit" directory is missing ! #'
	@echo '################################################################'
endif
	$(MAKE) test-ansible-sanity
ifneq ($(wildcard tests/integration/targets),) # Execute tests only if there is the required directory !
	$(MAKE) test-ansible-integration
else
	@echo '#######################################################################################'
	@echo '# Integration tests deactivated as "tests/integration/targets" directory is missing ! #'
	@echo '#######################################################################################'
endif

.PHONY: test-ansible-units
test-ansible-units: ## 🏃 Launch ansible unit tests
#### Use units_o="..." to specify options (--color, --docker, --coverage, etc)
$(eval units_o ?= -v --color --requirements $(ANSIBLE_COVERAGE_OPTION) $(ANSIBLE_DOCKER_OPTION) $(ANSIBLE_PY_TARGET_OPTION))
test-ansible-units:
	@echo "~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~"
	@echo "~                       ~~ Ansible unit tests ~~                                ~"
	@echo "~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~"
	cd ${COLLECTION_BUILD_DIR} && ansible-test units $(units_o)

.PHONY: test-ansible-integration
test-ansible-integration: ## 🏃 Launch ansible integration tests
#### Use integration_o="..." to specify options (--retry-on-error, --python VERSION, --docker, --coverage, etc)
$(eval integration_o ?= -v --color --requirements --diff --retry-on-error --continue-on-error $(ANSIBLE_COVERAGE_OPTION) $(ANSIBLE_DOCKER_OPTION) $(ANSIBLE_PY_TARGET_OPTION))
test-ansible-integration:
	@echo "~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~"
	@echo "~                    ~~ Ansible integration tests ~~                            ~"
	@echo "~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~"
	cd ${COLLECTION_BUILD_DIR} && ansible-test integration $(integration_o)

.PHONY: test-ansible-sanity
test-ansible-sanity: ## 🏃 Launch ansible sanity checks.
#### Use sanity_o="..." to specify options (--test TEST_NAME, --docker, --coverage, etc)
$(eval sanity_o ?= -v --color --requirements $(ANSIBLE_COVERAGE_OPTION) $(ANSIBLE_DOCKER_OPTION) $(ANSIBLE_PY_TARGET_OPTION))
test-ansible-sanity:
	@echo "~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~"
	@echo "~                      ~~ Ansible sanity tests ~~                               ~"
	@echo "~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~"
	cd ${COLLECTION_BUILD_DIR} && ansible-test sanity $(sanity_o)

.PHONY: test-ansible-coverage
test-ansible-coverage: ## 🏃 Launch ansible coverage generation ("report" by default, for console report)
#### Use coverage_o="..." to specify options (--requirements, --group-by GROUP, etc)
$(eval coverage_o ?= -v --color --requirements)
#### Use coverage_c="..." to specify coverage command (report, html, combine, etc)
ifeq ($(WITH_COVERAGE_REPORT),console)
$(eval coverage_c ?= report)
test-ansible-coverage: coverage_c?=report
else
$(eval coverage_c ?= $(WITH_COVERAGE_REPORT))
test-ansible-coverage: coverage_c?=$(WITH_COVERAGE_REPORT)
endif
test-ansible-coverage:
	cd ${COLLECTION_BUILD_DIR} && ansible-test coverage $(coverage_c) $(coverage_o);

##
##—— 🔧 Options ——————————————————————————————————————————————————————————————
##? Python binary 🐍      Use `PYTHON=... make TARGET` to override Python binary used (default to "python")
##? Build directory 🗜️     Use `BUILD_DIR=... make TARGET` to override build directory (default to "build" directory inside project)
##  🇦 Ansible
##? Docker 🐋             Use `WITH_DOCKER=1 make TARGET` to enable docker for ansible tests
##? Python target 🐍      Use `WITH_PY_TARGET=X.Y make TARGET` to target a specific python version during ansible tests
##? Coverage 📈           Use `WITH_COVERAGE=1 make TARGET` to enable coverage where it's doable
##? Coverage report 📝    Use `WITH_COVERAGE_REPORT=... make TARGET` to define default coverage command

