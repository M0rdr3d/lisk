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
	nvm(getNodejsVersion()) {
		sh '''
		dropdb --if-exists lisk_dev
		createdb lisk_dev
		NODE_ENV=test node app.js &>.app.log &
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

def teardown() {
	timeout(60) {
		sh 'killall --verbose --wait node || true'
	}
	archiveArtifacts artifacts: 'lisk_*.log', allowEmptyArchive: true
	deleteDir()
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
				stage('Functional HTTP GET tests') {
					agent { node { label 'lisk-core' } }
					steps {
						unstash 'build'
						setup()
						ansiColor('xterm') {
							timestamps {
								nvm(getNodejsVersion()) {
									sh '''
									if [ "$JENKINS_PROFILE" = "jenkins-extensive" ]; then
										npm test -- mocha:extensive:functional:get
										mv logs/devnet/lisk.log lisk_get_extensive.log
									else
										npm test -- mocha:default:functional:get
										mv logs/devnet/lisk.log lisk_get.log
									fi
									'''
								}
							}
						}
					}
					post {
						always {
							teardown()
						}
						failure {
							sh 'curl --verbose localhost:4000/api/node/constants |jq .'
						}
					}
				}
				stage('Functional HTTP POST tests') {
					agent { node { label 'lisk-core' } }
					steps {
						unstash 'build'
						setup()
						ansiColor('xterm') {
							timestamps {
								nvm(getNodejsVersion()) {
									sh '''
									if [ "$JENKINS_PROFILE" = "jenkins-extensive" ]; then
										npm test -- mocha:extensive:functional:post
										mv logs/devnet/lisk.log lisk_post_extensive.log
									else
										npm test -- mocha:default:functional:post
										mv logs/devnet/lisk.log lisk_post.log
									fi
									'''
								}
							}
						}
					}
					post {
						always {
							teardown()
						}
					}
				}
				stage ('Functional WS tests') {
					agent { node { label 'lisk-core' } }
					steps {
						unstash 'build'
						setup()
						ansiColor('xterm') {
							timestamps {
								nvm(getNodejsVersion()) {
									sh '''
									if [ "$JENKINS_PROFILE" = "jenkins-extensive" ]; then
										npm test -- mocha:extensive:functional:ws
										mv logs/devnet/lisk.log lisk_ws_extensive.log
									else
										npm test -- mocha:default:functional:ws
										mv logs/devnet/lisk.log lisk_ws.log
									fi
									'''
								}
							}
						}
					}
					post {
						always {
							teardown()
						}
					}
				}
				stage('Unit tests') {
					agent { node { label 'lisk-core' } }
					steps {
						unstash 'build'
						setup()
						ansiColor('xterm') {
							timestamps {
								nvm(getNodejsVersion()) {
									sh '''
									if [ "$JENKINS_PROFILE" = "jenkins-extensive" ]; then
										npm test -- mocha:extensive:unit
										mv logs/devnet/lisk.log lisk_unit_extensive.log
									else
										npm test -- mocha:default:unit
										mv logs/devnet/lisk.log lisk_unit.log
									fi
									'''
								}
							}
						}
					}
					post {
						always {
							teardown()
						}
					}
				}
				stage('Integation tests') {
					agent { node { label 'lisk-core' } }
					steps {
						unstash 'build'
						setup()
						ansiColor('xterm') {
							timeout(5) {
								timestamps {
									nvm(getNodejsVersion()) {
										sh '''
										if [ "$JENKINS_PROFILE" = "jenkins-extensive" ]; then
											npm test -- mocha:extensive:integration
											mv logs/devnet/lisk.log lisk_integration_extensive.log
										else
											npm test -- mocha:default:integration
											mv logs/devnet/lisk.log lisk_integration.log
										fi
										'''
									}
								}
							}
						}
					}
					post {
						always {
							teardown()
						}
					}
				}
			}
		}
		// TODO: coverage
	}
	post {
		failure {
			script {
				build_info = getBuildInfo()
				//liskSlackSend('danger', "Build ${build_info} failed (<${env.BUILD_URL}/console|console>, <${env.BUILD_URL}/changes|changes>)")
				sh 'TODO: activate slack notification'
			}
		}
	}
}
