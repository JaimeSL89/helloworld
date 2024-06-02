pipeline {
    agent any

    stages {
        stage('Get Code') {
            steps {
                //obtener el comando del repositorio
				git 'https://github.com/JaimeSL89/helloworld.git'
                stash name:'code', includes:'**'
            }
        }
        stage('Static') {
            steps {
                bat 'python -m flake8 --exit-zero --format=pylint app >flake8.out'
                recordIssues tools: [flake8(name: 'flake8', pattern: 'flake8.out')], qualityGates: [[threshold: 8, type: 'TOTAL', unstable: true],[threshold: 10, type: 'TOTAL', unstable: false]]
            }
        }
        stage('Security'){
            steps {
                bat 'python -m bandit --exit-zero -r . -f custom -o bandit.out --severity-level medium --msg-template "{abspath}:{line}: [{test_id}] {msg}"'
                recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')], qualityGates: [[threshold: 1, type: 'TOTAL', unstable: true], [threshold: 2, type: 'TOTAL', unstable: false]]
            }
        }
        stage('Performance') {
            steps {
                bat '''
                    set PYTHONPATH=.
                    set FLASK_APP=app/api.py
                    start python -m flask run
                    C:\\unir\\apache-jmeter-5.6.3\\bin\\jmeter -n -t test\\jmeter\\flask.jmx -f -l flask.jtl
                '''
                perfReport sourceDataFiles: 'flask.jtl'
            }
        }
        stage('Tests') {
            parallel{
                stage ('unit'){
                    //agent { label 'agent1' }
                        steps {
                            bat '''
                                whoami
                                hostname
                                dir
                            '''
                            //ejecucion de los test unitarios
                            echo 'Ejecutando los test unitarios'
                            catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE'){
                                unstash name:'code'
                                bat '''
                                    python --version
                                    dir
                                    set PYTHONPATH=.
                                    python -m pytest --junitxml=result-unit.xml test/unit
                                    
                                '''
                                stash name: 'unit-res', includes:'result-unit.xml'
                            }
                        }
                }
                stage('Rest') {
                    //agent { label 'agent2' }
                    //agent any
                        steps {
                            catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE'){
                                unstash name:'code'
                                bat '''
                                    whoami
                                    hostname
                                    dir
                                '''
                                //ejecucion de los test rest
                                echo 'Ejecutando los test Rest'
                                bat '''
                                    python --version
                                    set PYTHONPATH=.
                                    set FLASK_APP=app/api.py
                                    start python -m flask run
                                    '''
                                    bat'''
                                    start java -jar C:\\unir\\wiremock-standalone-3.5.4.jar --port 9090 --root-dir test\\wiremock
                                    '''
                                    sleep 15
                                    bat'''
                                    python -m pytest --junitxml=result-rest.xml test\\rest
                                    '''
                                    stash name: 'rest-res', includes:'result-rest.xml'
                                }
                            
                        }
                }
            }
        }
		stage('Cobertura') {
            steps {
              catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                bat '''
                    SET PYTHONPATH=%WORKSPACE%
                    python -m coverage run --branch --source=app --omit=app\\__init__.py,app\\api.py -m pytest test\\unit
                    python -m coverage xml
                '''
                cobertura coberturaReportFile: 'coverage.xml', failUnhealthy: true, failUnstable: false, conditionalCoverageTargets: '100,80,90', lineCoverageTargets: '100,85,95'

              }  
            }
        }
    }   
}
