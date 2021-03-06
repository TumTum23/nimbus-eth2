// https://stackoverflow.com/questions/40760716/jenkins-abort-running-build-if-new-one-is-started
// We should only abort older jobs in PR branches, so we have a nice CI history in "master" and "devel".
if (env.BRANCH_NAME != "master" && env.BRANCH_NAME != "devel") {
	def buildNumber = env.BUILD_NUMBER as int
	if (buildNumber > 1) {
		milestone(buildNumber - 1)
	}
	milestone(buildNumber)
}

def runStages() {
	try {
		stage("Clone") {
			/* source code checkout */
			checkout scm
			/* we need to update the submodules before caching kicks in */
			sh "git submodule update --init --recursive"
		}

		cache(maxCacheSize: 250, caches: [
			[$class: "ArbitraryFileCache", excludes: "", includes: "**/*", path: "${WORKSPACE}/vendor/nimbus-build-system/vendor/Nim/bin"],
			[$class: "ArbitraryFileCache", excludes: "", includes: "**/*", path: "${WORKSPACE}/jsonTestsCache"]
		]) {
			stage("Build") {
				sh """#!/bin/bash
				set -e
				make -j${env.NPROC} V=1 update # to allow a newer Nim version to be detected
				make -j${env.NPROC} V=1 deps # to allow the following parallel stages
				V=1 ./scripts/setup_official_tests.sh jsonTestsCache
				"""
			}
		}

		stage("Test") {
			parallel(
				"tools": {
					stage("Tools") {
						sh """#!/bin/bash
						set -e
						make -j${env.NPROC} V=1
						make -j${env.NPROC} V=1 LOG_LEVEL=TRACE NIMFLAGS='-d:testnet_servers_image' nimbus_beacon_node
            # Miracl fallback
            # make -j${env.NPROC} V=1 LOG_LEVEL=TRACE NIMFLAGS='-d:BLS_FORCE_BACKEND=miracl -d:testnet_servers_image' nimbus_beacon_node
						"""
					}
				},
				"test suite": {
					stage("Test suite") {
						sh "make -j${env.NPROC} V=1 DISABLE_TEST_FIXTURES_SCRIPT=1 test"
					}
					stage("testnet finalization") {
						// EXECUTOR_NUMBER will be 0 or 1, since we have 2 executors per Jenkins node
						sh """#!/bin/bash
						set -e
						./scripts/launch_local_testnet.sh --testnet 0 --nodes 4 --stop-at-epoch 5 --log-level DEBUG --disable-htop --enable-logtrace --data-dir local_testnet0_data --base-port \$(( 9000 + EXECUTOR_NUMBER * 100 )) --base-rpc-port \$(( 7000 + EXECUTOR_NUMBER * 100 )) --base-metrics-port \$(( 8008 + EXECUTOR_NUMBER * 100 )) -- --verify-finalization --discv5:no
						./scripts/launch_local_testnet.sh --testnet 1 --nodes 4 --stop-at-epoch 5 --log-level DEBUG --disable-htop --enable-logtrace --data-dir local_testnet1_data --base-port \$(( 9000 + EXECUTOR_NUMBER * 100 )) --base-rpc-port \$(( 7000 + EXECUTOR_NUMBER * 100 )) --base-metrics-port \$(( 8008 + EXECUTOR_NUMBER * 100 )) -- --verify-finalization --discv5:no
						"""
					}
				}
			)
		}
	} catch(e) {
		// we need to rethrow the exception here
		throw e
	} finally {
		// archive testnet logs
		sh """#!/bin/bash
		for D in local_testnet0_data local_testnet1_data; do
			[[ -d "\$D" ]] && tar cjf "\${D}-\${NODE_NAME}.tar.bz2" "\${D}"/*.txt || true
		done
		"""
		try {
			archiveArtifacts("*.tar.bz2")
		} catch(e) {
			println("Couldn't archive artefacts.")
			println(e.toString());
			// we don't need to re-raise it here; it might be a PR build being cancelled by a newer one
		}
		// clean the workspace
		cleanWs(disableDeferredWipeout: true, deleteDirs: true)
	}
}

parallel(
	"Linux": {
		node("linux") {
			withEnv(["NPROC=${sh(returnStdout: true, script: 'nproc').trim()}"]) {
				runStages()
			}
		}
	},
	"macOS": {
		node("macos") {
			withEnv(["NPROC=${sh(returnStdout: true, script: 'sysctl -n hw.logicalcpu').trim()}"]) {
				runStages()
			}
		}
	},
)
