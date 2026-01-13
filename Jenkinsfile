pipeline {
    agent any
    
    options { skipDefaultCheckout() }
    
    stages {
        stage('Get Code'){
            steps {
                git 'https://github.com/anieto-unir/helloworld.git'
                stash name:'code', includes:'*'
            }
        }
        stage('Tests') {
                parallel {
                    stage('Unit'){
                        steps {
                            catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                                unstash name:'code'
                                sh '''
                                    export PYTHONPATH="$PWD"
                                    pytest --junitxml=result-unit.xml test/unit
                                '''
                                stash name:'unit-res', includes:'result-unit.xml'
                            }
                        }
                    }
                    stage('Rest'){
                        steps {
                            catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                                unstash name:'code'
                                sh '''
                                    export FLASK_APP=app/api.py
                                    flask run &
                                    sleep 10
                                    java  -jar /var/lib/jenkins/wiremock-standalone-3.13.2.jar --port 9090 --root-dir test/wiremock/ &
                                    sleep 10
                                    export PYTHONPATH="$PWD"
                                    pytest --junitxml=result-rest.xml test/rest
                                '''
                                stash name:'rest-res', includes:'result-rest.xml'
                            }
                        }
                    }
                    stage('Static'){
                        steps {
                           sh 'flake8 --format=pylint --exit-zero app > flake8.out'
                           recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')], qualityGates: [[threshold:
                    8, type: 'TOTAL', unstable: true],[threshold:10, type: 'TOTAL', unstable: false]]
                        }
                    }
                    stage('Security'){
                        steps {
                            sh 'bandit --exit-zero -r . -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"'
                            recordIssues tools: [pyLint(name:'Bandit', pattern: 'bandit.out')], qualityGates: [[threshold:
                    2, type: 'TOTAL', unstable: true],[threshold:4, type: 'TOTAL', unstable: false]]
                        }
                    }
                    stage('Cobertura'){
                        steps {
                            sh '''
                                export PYTHONPATH="$PWD"
                                coverage run --branch --source=app --omit=app/__init__.py,app/api.py -m pytest test/unit
                                coverage xml
                            '''
                            recordCoverage qualityGates: [[criticality: 'ERROR', integerThreshold: 85, metric: 'LINE', threshold: 85.0], [criticality: 'NOTE', integerThreshold: 95, metric: 'LINE', threshold: 95.0], [criticality: 'ERROR', integerThreshold: 80, metric: 'BRANCH', threshold: 80.0], [criticality: 'NOTE', integerThreshold: 90, metric: 'BRANCH', threshold: 90.0]], tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']]
                        }
                    }
                }
        }
        stage('Performance') {
            steps {
                sh '''
                    export FLASK_APP=app/api.py
                    /usr/local/bin/apache-jmeter-5.6.3/bin/jmeter -n -t /usr/local/bin/apache-jmeter-5.6.3/ThreadGroup.jmx -f -l flask.jtl
                '''
                perfReport sourceDataFiles: 'flask.jtl'
            }
        }
        stage('Results') {
            steps {
                unstash name:'unit-res'
                unstash name:'rest-res'
                junit 'result*.xml'
            }
        }
    }
}
