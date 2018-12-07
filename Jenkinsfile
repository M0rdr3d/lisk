/*
 * Copyright © 2018 Lisk Foundation
 *
 * See the LICENSE file at the top-level directory of this distribution
 * for licensing information.
 *
 * Unless otherwise agreed in a custom licensing agreement with the Lisk Foundation,
 * no part of this software, including this file, may be copied, modified,
 * propagated, or distributed except according to the terms contained in the
 * LICENSE file.
 *
 * Removal or modification of this copyright notice is prohibited.
 */

@Library('lisk-jenkins') _

def setup() {
	cleanWs()
	unstash 'build'
	nvm(getNodejsVersion()) {
		sh '''
		# shouldn't hurt assuming the 'lisk-core' jenkins nodes have 1 executor
		killall --verbose --wait node || true
		dropdb --if-exists lisk_dev
		createdb lisk_dev
		NODE_ENV=test node app.js >.app.log 2>&1 &
		'''
	}
	timeout(1) {
		waitUntil {
			script {
				def api_available = sh script: 'curl --silent http://localhost:4000/api/node/constants >/dev/null', returnStatus: true
				return (api_available == 0)
			}
		}
	}
}

def teardown(test_name) {
	nvm(getNodejsVersion()) {
		sh '''
		npm run cover:report
		HOST=localhost:4000 npm run cover:fetch
		'''
	}
	sh """
	rm -rf coverage_${test_name}
	mkdir -p coverage_${test_name}
	mv test/.coverage-unit/* test/.coverage-func.zip coverage_${test_name}/
	"""
	stash name: "coverage_${test_name}", includes: "coverage_${test_name}/"
	timeout(1) {
		sh 'killall --verbose --wait node || true'
	}
	archiveArtifacts artifacts: 'lisk_*.log', allowEmptyArchive: true
	archiveArtifacts artifacts: 'test_*.log', allowEmptyArchive: true
	cleanWs()
}

def report_coverage() {
	sh '''
	rm -rf coverage
	mkdir -p coverage
	'''
	dir('coverage') {
		['get', 'post', 'ws', 'unit', 'integration'].each {
			try {
				unstash "coverage_${it}"
				sh "unzip coverage_${it}/.coverage-func.zip -d coverage_${it}/func/"
			} catch(err) {
				println "Could not unstash ${it}. Continuing."
			}
		}
		sh '''
		find -name lcov.info |sed 's/^/-a /' |xargs lcov -o merged.lcov
		sed -i -r -e "s#$WORKSPACE##g" merged.lcov
		cp ~/.core_coveralls.yml .coveralls.yml
		'''
		nvm(getNodejsVersion()) {
			sh '../node_modules/.bin/coveralls <merged.lcov'
		}
	}
}


properties([
	parameters([
		string(name: 'JENKINS_PROFILE', defaultValue: 'jenkins', description: 'To build cache dependencies and run slow tests, change this value to jenkins-extensive.', ),
		string(name: 'LOG_LEVEL', defaultValue: 'error', description: 'To get desired build log output change the log level', ),
		string(name: 'FILE_LOG_LEVEL', defaultValue: 'error', description: 'To get desired file log output change the log level', ),
		string(name: 'LOG_DB_EVENTS', defaultValue: 'false', description: 'To get detailed info on db events log.', ),
		string(name: 'SILENT', defaultValue: 'true', description: 'To turn off test debug logs.', )
	 ])
])


pipeline {
	agent { node { label 'lisk-core' } }

	stages {
		stage('Build') {
			steps {
				nvm(getNodejsVersion()) {
					sh 'npm ci'
				}
				stash name: 'build'
			}
		}
		stage('Parallel: tests wrapper') {
			parallel {
				stage('Linter') {
					steps {
						node('lisk-core') {
							unstash 'build'
							nvm(getNodejsVersion()) {
								sh 'npm run lint'
							}
						}
					}
				}
				stage('Functional HTTP POST tests') {
					agent { node { label 'lisk-core' } }
					steps {
						setup()
						ansiColor('xterm') {
							timestamps {
								nvm(getNodejsVersion()) {
									sh '''
									if [ "$JENKINS_PROFILE" = "jenkins-extensive" ]; then
										npm test -- mocha:extensive:functional:post
										mv logs/devnet/lisk.log test_post_extensive.log
									else
										npm test -- mocha:default:functional:post
										mv logs/devnet/lisk.log test_post.log
									fi
									mv .app.log lisk_post.log
									'''
								}
							}
						}
					}
					post {
						cleanup {
							teardown('post')
						}
					}
				}
				stage ('Functional WS tests') {
					agent { node { label 'lisk-core' } }
					steps {
						setup()
						ansiColor('xterm') {
							timestamps {
								nvm(getNodejsVersion()) {
									sh '''
									if [ "$JENKINS_PROFILE" = "jenkins-extensive" ]; then
										npm test -- mocha:extensive:functional:ws
										mv logs/devnet/lisk.log test_ws_extensive.log
									else
										npm test -- mocha:default:functional:ws
										mv logs/devnet/lisk.log test_ws.log
									fi
									mv .app.log lisk_ws.log
									'''
								}
							}
						}
					}
					post {
						cleanup {
							teardown('ws')
						}
					}
				}
				stage('Unit tests') {
					agent { node { label 'lisk-core' } }
					steps {
						setup()
						ansiColor('xterm') {
							timestamps {
								nvm(getNodejsVersion()) {
									sh '''
									if [ "$JENKINS_PROFILE" = "jenkins-extensive" ]; then
										npm test -- mocha:extensive:unit
										mv logs/devnet/lisk.log test_unit_extensive.log
									else
										npm test -- mocha:default:unit
										mv logs/devnet/lisk.log test_unit.log
									fi
									mv .app.log lisk_unit.log
									'''
								}
							}
						}
					}
					post {
						cleanup {
							teardown('unit')
						}
					}
				}
			}
		}
	}
	post {
		always {
			report_coverage()
		}
		failure {
			script {
				build_info = getBuildInfo()
				//liskSlackSend('danger', "Build ${build_info} failed (<${env.BUILD_URL}/console|console>, <${env.BUILD_URL}/changes|changes>)")
			}
		}
		cleanup {
			cleanWs()
		}
	}
}
