pipeline {
  agent none

  environment {
    FLASK_APP = 'app/api.py'
  }

  stages {

    stage('Get Code') {
      agent { label 'build-agent' }
      steps {
        sh 'whoami && hostname && echo ${WORKSPACE}'
        checkout scm
      }
    }

    stage('Dependencies and Wiremock') {
      parallel {

        stage('Dependencies') {
          agent { label 'build-agent' }
          steps {
            sh 'whoami && hostname && echo ${WORKSPACE}'
            sh '''
              python3 -m venv venv
              ./venv/bin/pip install --upgrade pip
              ./venv/bin/pip install -r requirements.txt
            '''
            stash name: 'venv', includes: 'venv/**'
          }
        }

        stage('Wiremock') {
          agent { label 'build-agent' }
          steps {
            sh 'whoami && hostname && echo ${WORKSPACE}'
            sh '''
              echo "Iniciando Wiremock en master"
              docker start wiremock || true
              sleep 4
            '''
          }
        }

      }
    }

    stage('Tests') {
      parallel {

        stage('Unit') {
          agent { label 'build-agent' }
          steps {
            sh 'whoami && hostname && echo ${WORKSPACE}'
            sh '''
              export PYTHONPATH=$PWD
              ./venv/bin/coverage run --branch --source=app \
                --omit=app/__init__.py,app/api.py \
                -m pytest test/unit --junitxml=result_unit.xml
              ./venv/bin/coverage xml -o coverage.xml
            '''
            junit 'result_unit.xml'
          }
        }

        stage('Rest') {
          agent { label 'build-agent' }
          steps {
            sh 'whoami && hostname && echo ${WORKSPACE}'
            sh '''
              export PYTHONPATH=$PWD
              ./venv/bin/flask run & sleep 3
              ./venv/bin/pytest test/rest --junitxml=result_rest.xml
            '''
            junit 'result_rest.xml'
          }
        }
        stage('Static Analysis') {
          agent { label 'analysis-agent' }
          steps {
            unstash 'venv'
            sh 'whoami && hostname && echo ${WORKSPACE}'
            sh '''
              ./venv/bin/flake8 --exit-zero --format=pylint app > flake8.out
              ./venv/bin/bandit --exit-zero -r app -f custom -o bandit.out
            '''
            recordIssues( 
              qualityGates: [
                [criticality: 'NOTE', integerThreshold: 8, threshold: 8.0, type: 'TOTAL'],
                [criticality: 'ERROR', integerThreshold: 10, threshold: 10.0, type: 'TOTAL']
              ],
              tools: [flake8(pattern: 'flake8.out')
              ]
            )
            recordIssues (
              qualityGates: [
                [criticality: 'NOTE', integerThreshold: 2, threshold: 2.0, type: 'TOTAL'],
                [criticality: 'FAILURE', integerThreshold: 4, threshold: 4.0, type: 'TOTAL']
              ], 
            tools: [pyLint(pattern: 'bandit.out')]
            )
          }
          post {
            always {
              cleanWs()
            }
          }
        }

      }
    }

    stage('Performance') {
      agent { label 'analysis-agent' }
      steps {
        unstash 'venv'
        sh 'whoami && hostname && echo ${WORKSPACE}'
        sh '''
          /opt/jmeter/bin/jmeter -n \
            -t test/jmeter/flask.jmx \
            -l flask.jtl -f
        '''
        perfReport sourceDataFiles: 'flask.jtl'
      }
      post {
        always {
          cleanWs()
        }
      }
    }

    stage('Coverage') {
      agent { label 'build-agent' }
      steps {
        sh 'whoami && hostname && echo ${WORKSPACE}'
        recordCoverage(
          tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']],
          qualityGates: [
              [criticality: 'ERROR', integerThreshold: 85, metric: 'LINE',   threshold: 85.0],
              [criticality: 'NOTE',  integerThreshold: 95, metric: 'LINE',   threshold: 95.0],
              [criticality: 'ERROR', integerThreshold: 80, metric: 'BRANCH', threshold: 80.0],
              [criticality: 'NOTE',  integerThreshold: 90, metric: 'BRANCH', threshold: 90.0]
          ]
        )
      }
    }

  }
}