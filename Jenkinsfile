pipeline {
  agent none

  environment {
    FLASK_APP = 'app/api.py'
  }

  stages {

    stage('Get Code') {
      agent { label 'master' }
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
          }
        }

        stage('Wiremock') {
          agent { label 'master' }
          steps {
            sh 'whoami && hostname && echo ${WORKSPACE}'
            sh '''
              echo "Arrancando Wiremock en master"
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

      }
    }

    stage('Static Analysis') {
      agent { label 'analysis-agent' }
      steps {
        sh 'whoami && hostname && echo ${WORKSPACE}'
        sh '''
          ./venv/bin/flake8 --exit-zero --format=pylint app > flake8.out
        '''
        recordIssues tools: [flake8(pattern: 'flake8.out')]
      }
    }

    stage('Security') {
      agent { label 'analysis-agent' }
      steps {
        sh 'whoami && hostname && echo ${WORKSPACE}'
        sh '''
          ./venv/bin/bandit --exit-zero -r app -f custom -o bandit.out
        '''
        recordIssues tools: [pyLint(pattern: 'bandit.out')]
      }
    }

    stage('Performance') {
      agent { label 'analysis-agent' }
      steps {
        sh 'whoami && hostname && echo ${WORKSPACE}'
        sh '''
          /opt/jmeter/bin/jmeter -n \
            -t test/jmeter/flask.jmx \
            -l flask.jtl -f
        '''
        perfReport sourceDataFiles: 'flask.jtl'
      }
    }

    stage('Coverage') {
      agent { label 'build-agent' }
      steps {
        sh 'whoami && hostname && echo ${WORKSPACE}'
        recordCoverage tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']]
      }
    }

    stage('Results') {
      agent { label 'build-agent' }
      steps {
        sh 'whoami && hostname && echo ${WORKSPACE}'
        junit 'result*.xml'
      }
    }

  }

  post {
    always {
      cleanWs()
    }
  }
}