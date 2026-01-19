pipeline {
  agent any
  environment {
        FLASK_APP = 'app/api.py'
  }
  stages {
    stage('Get Code') {
      steps {
        checkout scm
      }
    }
    stage('Dependencies and Wiremock') {
      parallel{
          stage('Dependencies') {
            steps {
              catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                  sh '''
                    python3 -m venv venv
                    ./venv/bin/pip install --upgrade pip
                    ./venv/bin/pip install -r requirements.txt
                  '''


              }
            }
          }
          stage('Wiremock') {
            steps {
              catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                  sh '''
                    docker start wiremock || true
                    sleep 4
                  '''
              }
            }
          }
      }
    }
     stage('Test') {
      // Se ejecutan las pruebas en un solo stage pero en paralelo
        parallel {
          stage('Unit') {
            steps {
              catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                sh '''
                  export PYTHONPATH=$PWD
                  ./venv/bin/pytest --junitxml=result_unit.xml test/unit
                '''
                junit 'result_unit.xml'
              }
            }
          }
          stage('Rest') {
            steps {
              catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                sh '''
                  export PYTHONPATH=$PWD
                  ./venv/bin/flask run & sleep 3
                  ./venv/bin/pytest --junitxml=result_rest.xml test/rest
                '''
                junit 'result_rest.xml'
              }
            }
          }
        }
    }
    stage('Static') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          sh '''
             ./venv/bin/flake8 --exit-zero --format=pylint app >flake8.out
          '''
          recordIssues(
            qualityGates: [
              [criticality: 'NOTE', integerThreshold: 8, threshold: 8.0, type: 'TOTAL'],
              [criticality: 'ERROR', integerThreshold: 10, threshold: 10.0, type: 'TOTAL']
            ],
            tools: [
              flake8(pattern: 'flake8.out')
            ]
          )
        }
      }
    }
     stage('Security') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          sh '''
            ./venv/bin/bandit --exit-zero -r app -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
          '''
          recordIssues(
           
            qualityGates: [
              [criticality: 'NOTE', integerThreshold: 2, type: 'TOTAL'],
              [criticality: 'FAILURE', integerThreshold: 4, type: 'TOTAL']
            ],
            tools: [
              pyLint(pattern: 'bandit.out')
            ]
          )
        }
      }
    }
    stage('Performance') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          sh '''
            echo "Verificando JMeter"
            /opt/jmeter/bin/jmeter -n -t test/jmeter/flask.jmx -l flask.jtl -f
          '''
          perfReport sourceDataFiles: 'flask.jtl'
        }
      }
    }
    stage('Coverage') {
      steps {
        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
          sh '''
              ./venv/bin/coverage run --branch --source=app --omit=app/__init__.py,app/api.py -m pytest test/unit --junitxml=result_unit.xml
              ./venv/bin/coverage xml -o coverage.xml
            '''
            recordCoverage(
              qualityGates: [
                [criticality: 'ERROR', integerThreshold: 85, metric: 'LINE', threshold: 85.0],
                [criticality: 'NOTE', integerThreshold: 95, metric: 'LINE', threshold: 95.0],
                [criticality: 'ERROR', integerThreshold: 80, metric: 'BRANCH', threshold: 80.0],
                [criticality: 'NOTE', integerThreshold: 90, metric: 'BRANCH', threshold: 90.0]
              ],
              tools: [
                [parser: 'COBERTURA', pattern: 'coverage.xml']
              ]
            )
          }
      }
    }
    stage('Results') {
      steps {
        junit 'result*.xml'
      }
    }
    
    
  
  }
}
