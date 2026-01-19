pipeline {
  agent none

  environment {
    FLASK_APP = 'app/api.py'
  }

  stages {
    stage('Dependencies and Wiremock') {
      parallel {

        stage('Dependencies') {
          agent { docker { image 'python:3.11' } }
          steps {
            sh 'id'
            sh 'hostname'
            sh 'echo ${WORKSPACE}'

            

            sh '''
              python3 -m venv venv
              ./venv/bin/pip install --upgrade pip
              ./venv/bin/pip install -r requirements.txt
            '''

            stash name: 'venv', includes: 'venv/**'
          }
        }

        stage('Wiremock') {
          agent { docker { image 'python:3.11' } }
          steps {
            sh 'id'
            sh 'hostname'
            sh 'echo ${WORKSPACE}'

            sh '''
              docker start wiremock || true
              sleep 4
            '''
          }
        }
      }
    }

    /* =======================
       TESTS (paralelos)
       ======================= */
    stage('Tests') {
      parallel {

        stage('Unit Tests') {
        agent { docker { image 'python:3.11' } }
          steps {
            sh 'id'
            sh 'hostname'
            sh 'echo ${WORKSPACE}'

            
            unstash 'venv'

            sh '''
              export PYTHONPATH=$PWD
              ./venv/bin/coverage run --branch --source=app --omit=app/__init__.py,app/api.py -m pytest test/unit --junitxml=result_unit.xml
              ./venv/bin/coverage xml -o coverage.xml
            '''
            junit 'result_unit.xml'
            stash name: 'coverage', includes: 'coverage.xml'
          }
        }

        stage('REST Tests') {
        agent { docker { image 'python:3.11' } }
          steps {
            sh 'id'
            sh 'hostname'
            sh 'echo ${WORKSPACE}'

            
            unstash 'venv'

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

    /* =======================
       STATIC ANALYSIS
       ======================= */
    stage('Static Analysis') {
      agent { docker { image 'python:3.11' } }
      steps {
        sh 'id'
        sh 'hostname'
        sh 'echo ${WORKSPACE}'

        
        unstash 'venv'

        sh '''
          ./venv/bin/flake8 --exit-zero --format=pylint app > flake8.out
        '''

        recordIssues(
          qualityGates: [
            [criticality: 'NOTE', integerThreshold: 8, type: 'TOTAL'],
            [criticality: 'ERROR', integerThreshold: 10, type: 'TOTAL']
          ],
          tools: [flake8(pattern: 'flake8.out')]
        )
      }
    }

    /* =======================
       SECURITY
       ======================= */
    stage('Security') {
      agent { docker { image 'python:3.11' } }
      steps {
        sh 'id'
        sh 'hostname'
        sh 'echo ${WORKSPACE}'

        
        unstash 'venv'

        sh '''
          ./venv/bin/bandit --exit-zero -r app -f custom -o bandit.out \
          --msg-template "{abspath}:{line}: [{test_id}] {msg}"
        '''

        recordIssues(
          qualityGates: [
            [criticality: 'NOTE', integerThreshold: 2, type: 'TOTAL'],
            [criticality: 'FAILURE', integerThreshold: 4, type: 'TOTAL']
          ],
          tools: [pyLint(pattern: 'bandit.out')]
        )
      }
    }

    /* =======================
       PERFORMANCE (SECUENCIAL)
       ======================= */
    stage('Performance') {
      agent { docker { image 'python:3.11' } }
      steps {
        sh 'id'
        sh 'hostname'
        sh 'echo ${WORKSPACE}'

        

        sh '''
          /opt/jmeter/bin/jmeter -n \
          -t test/jmeter/flask.jmx \
          -l flask.jtl -f
        '''

        perfReport sourceDataFiles: 'flask.jtl'
      }
    }

    /* =======================
       COVERAGE
       ======================= */
    stage('Coverage') {
      agent { docker { image 'python:3.11' } }
      steps {
        sh 'id'
        sh 'hostname'
        sh 'echo ${WORKSPACE}'

        
        unstash 'coverage'
        unstash 'venv'

        recordCoverage(
          qualityGates: [
            [criticality: 'ERROR', integerThreshold: 85, metric: 'LINE'],
            [criticality: 'NOTE', integerThreshold: 95, metric: 'LINE'],
            [criticality: 'ERROR', integerThreshold: 80, metric: 'BRANCH'],
            [criticality: 'NOTE', integerThreshold: 90, metric: 'BRANCH']
          ],
          tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']]
        )
      }
    }
  }

  post {
    always {
      cleanWs()
    }
  }
}