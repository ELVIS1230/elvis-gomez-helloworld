pipeline {
  agent any
  environment {
    FLASK_APP = 'app/api.py'
  }
  stages {
    stage('Dependencies') {
      steps {
        sh '''
          python3 -m venv venv
          ./venv/bin/pip install --upgrade pip
          ./venv/bin/pip install -r requirements.txt
        '''
      }
    }
    stage('Static') {
      steps {
        sh '''
          ./venv/bin/flake8 --exit-zero --format=pylint app >flake8.out
        '''
        recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')], qualityGates: [
          [threshold: 16, type: 'TOTAL', unstable: true],
          [threshold: 17, type: 'TOTAL', unstable: false]
        ]
      }
    }
    stage('Test') {
      parallel {
        stage('Unit') {
          steps {
            catchError(buildResult: 'Unstable', stageResult: 'FAILURE') {
              sh '''
                export PYTHONPATH=$PWD
                ./venv/bin/pytest --junitxml=result-unit.xml test/unit
              '''
            }
          }
        }
        stage('Service') {
          steps {
            sh '''
              export PYTHONPATH=$PWD
              ./venv/bin/flask run & sleep 3
              ./venv/bin/pytest --junitxml=result.xml test/rest
            '''
          }
        }
      }
    }
    stage('Results') {
      steps {
        junit 'result*.xml'
      }
    }
    stage('Cobertura') {
      steps {
        sh '''
          ./venv/bin/coverage run --branch --source=app --omit=app/__init__.py,app/api.py -m pytest test/unit
          ./venv/bin/coverage xml
        '''
        recordCoverage(
          qualityGates: [
            [criticality: 'NOTE', integerThreshold: 85, metric: 'LINE', threshold: 85.0],
            [criticality: 'ERROR', integerThreshold: 60, metric: 'LINE', threshold: 60.0],
            [criticality: 'NOTE', metric: 'BRANCH']
          ],
          tools: [
            [parser: 'COBERTURA', pattern: 'coverage.xml']
          ]
        )
      }
    }
    stage('Security') {
      steps {
        sh '''
          ./venv/bin/bandit --exit-zero -r . -f custom -o bandit.out --msg-template "{abspath}:{line} [{test_id} {msg}]"
        '''
        recordIssues(
          tools: [
            pyLint(name: 'Bandit', pattern: 'bandit.out')
          ],
          qualityGates: [
            [threshold: 1, type: 'TOTAL', unstable: true],
            [threshold: 2, type: 'TOTAL', unstable: false]
          ]
        )
      }
    }
    stage('Performance') {
      steps {
        sh '''
          echo "Verificando JMeter"
          /opt/jmeter/bin/jmeter -n -t test/jmeter/flask.jmx -l flask.jtl -f
        '''
        perfReport sourceDataFiles: 'flask.jtl'
      }
    }
  }
}
