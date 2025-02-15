## This Source Code Form is subject to the terms of the Mozilla Public
## License, v. 2.0. If a copy of the MPL was not distributed with this
## file, You can obtain one at https://mozilla.org/MPL/2.0/.

.ONESHELL:
.SHELLFLAGS := -euc
MAKEFLAGS += --no-builtin-rules --no-builtin-variables
unexport MAKEFLAGS
export JOBS = $(patsubst -j%,%,$(filter -j%,${MAKEFLAGS}))

##	Set SHVERBOSE to make the shell verbose about command execution.
ifdef SHVERBOSE
.SHELLFLAGS := -x ${.SHELLFLAGS}
endif

## TODO Rebuild when any of the versions change.
vDesktop := 2.26.1
vCoreLib := 3.5.4
vCoreNPM := 6.11.3

dirRoot := ${PWD}
dirDesktop := ${dirRoot}/desktop
dirCore := ${dirRoot}/core

coreLibSrc=/core/src
dirCoreLib := ${dirCore}/lib
dirCoreLibSrc := ${dirCoreLib}${coreLibSrc}
dirCoreLibBuild := ${dirCoreLib}/build
dirCoreLibDist := ${dirCoreLibBuild}${coreLibSrc}

dirCoreNPM := ${dirCore}/node-bindings

artifact := ${dirDesktop}/dist/ledger-live-desktop-${vDesktop}-linux-arm64.AppImage
coreLib := ${dirCoreLibDist}/libledger-core.so
nodePlugin := ${dirCoreNPM}/build/Release/ledger-core-node.node

desktopGit := ${dirDesktop}/.git/HEAD
coreLibGit := ${dirCoreLib}/.git/HEAD
coreNPMGit := ${dirCoreNPM}/.git/HEAD

dep := ${dirRoot}/.dep/

##	If a patch cannot be reversed, it MUST apply, or things are broken.
define patch
if ! git -C "${1}" apply --check --reverse "${2}" 2> /dev/null; then
	git -C "${1}" apply "${2}"
fi
endef


.PHONY: desktop coreNPM coreLib cleanDeps

desktop: ${artifact}
##	Ledger Live desktop application.
${artifact}: ${desktopGit} ${dep}nodePluginPrereqs
##	HACK Hardcoded architecture to ARM64. Needs something more flexible.
	$(call patch,${dirDesktop},${dirRoot}/desktop-electron-builder-x64-to-arm64.patch)

	cd ${dirDesktop}
##	Force Yarn to use local core lib and bindings instead of a download.
	sed -Ei '\#^\s+"@ledgerhq/ledger-core":\s+"${vCoreNPM}",$$#s#${vCoreNPM}#file:${dirCoreNPM}#' \
		package.json

	yarn
##	--dirty to avoid double build (above needed for dist to work at all).
	yarn dist --dirty

##	Cast Ye thyne arcane Spelle to unset single letter flags from MFLAGS.
unFlag = $(strip $(shell printf '%s' $(firstword ${2})|sed -E 's/^(-[^${1}-]*)[${1}]+/\1/;/^-$$/d') $(wordlist 2,$(words ${2}),${2}))
##	TODO Figure out whether we need it expanding recursively or simply.
mFlags = $(call unFlag,rR,${MFLAGS})

coreLib: ${coreLib}
##	Ledger Live core library.
${coreLib}: ${dep}coreLibCmake
	cd ${dirCoreLibBuild}
	$(MAKE) ${mFlags} --no-print-directory

.PRECIOUS: ${coreLib}

coreNPM: ${nodePlugin}
##	Nodejs bindings for the core library.
${nodePlugin}: ${dep}nodePluginPrereqs
	cd ${dirCoreNPM}
	yarn

# ${dirCoreNPM}/build/Makefile
# $(MAKE) ${mFlags} -C ${dirCoreNPM}/build

.PRECIOUS: ${nodePlugin}

##	Purge dependency marker/sentinel files.
cleanDeps:
	rm -rf ${dep}


##	"Mark" a dependency.
define mark
	mkdir -p $(dir ${1})
	touch ${1}
endef

##	Dependency for building the core lib node bindings.
${dep}nodePluginPrereqs: ${coreLib} ${dep}djinni ${dep}coreNPMpkgJson
	$(call mark,$@)

##	Core lib cmake dependency.
${dep}coreLibCmake: ${dep}coreLibPatch
	mkdir -p ${dirCoreLibBuild}
	cd ${dirCoreLibBuild}

	cmake ${dirCoreLib}
	$(call mark,$@)

##	Djinni generation of core lib node binding sources.
${dep}djinni: ${coreLib} ${dep}genBindingsScript ${coreNPMGit}
	cd ${dirCoreLib}/djinni
	git fetch origin --prune
	git rebase origin/master

	cd ${dirCoreLib}
	tools/generateBindings.sh ${dirCoreNPM} ${dirCoreLibBuild}

##	XXX Misconfiguration(?) prevents Djinni from replacing these files.
	rm ${dirCoreNPM}/src/NJSPreferencesBackendCpp.?pp 2> /dev/null || true
##	As advised by the core lib README, remove the files generated there.
	git checkout -- core/src

	$(call mark,$@)

${dirCoreNPM}/package.json: ${coreNPMGit}
##	Indirect dependency, rebuilt if the file changes.
${dep}coreNPMpkgJson: ${dirCoreNPM}/package.json
##	Prevent preinstaller replacing the built core lib with a download.
	sed -i '/"preinstall":.*preinstall.js"/d' $<
	$(call mark,$@)

##	Patch the core lib to compile on ARM architectures.
##	XXX Remove this once LedgerHQ merges PR #764 (or an analogous fix).
${dep}coreLibPatch: ${dep}coreLibSrc
	$(call patch,${dirCoreLib},${dirRoot}/core-lib-sha512256-portability-arm64.patch)
	$(call mark,$@)

${dirCoreLib}/tools/generateBindings.sh: ${coreLibGit}
##	Indirect dependency, rebuilt if the file changes.
${dep}genBindingsScript: ${dirCoreLib}/tools/generateBindings.sh
##	Patch that script to abort on error.
	sed -i '1,/^#!/s/bash$$/-S bash -e/' $<
	$(call mark,$@)

##	Find core lib sources excluding Djinni-generated directory contents.
define findCoreSrc
find ${1} \
	-type d \( \
		-path ${1}/api -or \
		-path ${1}/jni \
	\) -prune -or \
	-type f -print
endef

##	File system dependency on core lib sources (without Djinni output).
##	BUG Can only ever findCoreSrc AFTER ${coreLibGit} is satisfied: parallel execution is flaky.
${dep}coreLibSrc: ${coreLibGit} $(shell $(call findCoreSrc,${dirCoreLibSrc}))
	$(call mark,$@)


${desktopGit}:
	git clone https://github.com/LedgerHQ/ledger-live-desktop.git ${dirDesktop}
	git -C ${dirDesktop} checkout v${vDesktop}

gitCoreLib := git -C ${dirCoreLib}
${coreLibGit}:
	git clone --recurse-submodules https://github.com/LedgerHQ/lib-ledger-core.git ${dirCoreLib}
	${gitCoreLib} checkout ${vCoreLib}
##	TODO Wtf is going on with libuv?
#	${gitCoreLib} submodule init
	${gitCoreLib} submodule update

${coreNPMGit}:
	git clone https://github.com/LedgerHQ/lib-ledger-core-node-bindings.git ${dirCoreNPM}
	git -C ${dirCoreNPM} checkout ${vCoreNPM}


##	An empty rule to avoid implicit rule search.
Makefile: ;
