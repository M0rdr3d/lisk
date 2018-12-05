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

// TODO: merge into startLisk
def initializeNode() {
	sh '''
	dropdb --if-exists lisk_dev
	createdb lisk_dev
	'''
}

def startLisk() {
	nvm(getNodejsVersion()) {
		sh 'NODE_ENV=test JENKINS_NODE_COOKIE=dontKillMe nohup node app.js &> .app.log &'
	}
	// TODO: use waitUntil instead
	sleep time: 15
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

node('lisk-core') {
	stage('Build') {
		checkout scm
		nvm(getNodejsVersion()) {
			sh 'npm ci'
		}
		stash name: 'build'
	}
	stage('Run parallel tests') {
		parallel(
			"Lint" : {
				node('lisk-core') {
					unstash 'build'
					timestamps {
						nvm(getNodejsVersion()) {
							sh 'npm run lint'
						}
					}
				}
			},
			"Functional HTTP GET tests" : {
				node('lisk-core') {
					unstash 'build'
					initializeNode()
					startLisk()
					ansiColor('xterm') {
						timestamps {
							nvm(getNodejsVersion()) {
								if (params.JENKINS_PROFILE == 'jenkins-extensive') {
									sh '''
									npm test -- mocha:extensive:functional:get
									mv logs/devnet/lisk.log lisk_get_extensive.log
									'''
								} else {
									sh '''
									npm test -- mocha:default:functional:get
									mv logs/devnet/lisk.log lisk_get.log
									'''
								}
							}
						}
					}
					archiveArtifacts artifacts: 'lisk_*.log', allowEmptyArchive: true
				}
			},
			"Functional HTTP POST tests" : {
				node('lisk-core') {
					unstash 'build'
					initializeNode()
					startLisk()
					ansiColor('xterm') {
						timestamps {
							nvm(getNodejsVersion()) {
								if (params.JENKINS_PROFILE == 'jenkins-extensive') {
									sh '''
									npm test -- mocha:extensive:functional:post
									mv logs/devnet/lisk.log lisk_post_extensive.log
									'''
								} else {
									sh '''
									npm test -- mocha:default:functional:post
									mv logs/devnet/lisk.log lisk_post.log
									'''
								}
							}
						}
					}
					archiveArtifacts artifacts: 'lisk_*.log', allowEmptyArchive: true
				}
			},
			"Functional WS tests" : {
				node('lisk-core') {
					unstash 'build'
					initializeNode()
					startLisk()
					ansiColor('xterm') {
						timestamps {
							nvm(getNodejsVersion()) {
								if (params.JENKINS_PROFILE == 'jenkins-extensive') {
									sh '''
									npm test -- mocha:extensive:functional:ws
									mv logs/devnet/lisk.log lisk_ws_extensive.log
									'''
								} else {
									sh '''
									npm test -- mocha:default:functional:ws
									mv logs/devnet/lisk.log lisk_ws.log
									'''
								}
							}
						}
					}
					archiveArtifacts artifacts: 'lisk_*.log', allowEmptyArchive: true
				}
			},
			"Unit tests" : {
				node('lisk-core') {
					unstash 'build'
					initializeNode()
					startLisk()
					ansiColor('xterm') {
						timestamps {
							nvm(getNodejsVersion()) {
								if (params.JENKINS_PROFILE == 'jenkins-extensive') {
									sh '''
									npm test -- mocha:extensive:unit
									mv logs/devnet/lisk.log lisk_unit_extensive.log
									'''
								} else {
									sh '''
									npm test -- mocha:default:unit
									mv logs/devnet/lisk.log lisk_unit.log
									'''
								}
							}
						}
					}
					archiveArtifacts artifacts: 'lisk_*.log', allowEmptyArchive: true
				}
			},
			"Integation tests" : {
				node('lisk-core') {
					unstash 'build'
					initializeNode()
					startLisk()
					ansiColor('xterm') {
						timestamps {
							nvm(getNodejsVersion()) {
								if (params.JENKINS_PROFILE == 'jenkins-extensive') {
									sh '''
									npm test -- mocha:extensive:integration
									mv logs/devnet/lisk.log lisk_integration_extensive.log
									'''
								} else {
									sh '''
									npm test -- mocha:default:integration
									mv logs/devnet/lisk.log lisk_integration.log
									'''
								}
							}
						}
					}
					archiveArtifacts artifacts: 'lisk_*.log', allowEmptyArchive: true
				}
			}
		)
	}
	// TODO: coverage
}
