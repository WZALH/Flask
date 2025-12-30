pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    VENV_DIR = ".venv"
    PYTHON   = "/usr/bin/python3"   
    APP_PORT = "5000"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD'
        sh 'ls -la'
      }
    }

    stage('Setup venv + Install deps') {
      steps {
        sh '''
          set -euxo pipefail
          $PYTHON -V

          if [ ! -d "$VENV_DIR" ]; then
            $PYTHON -m venv "$VENV_DIR"
          fi

          . "$VENV_DIR/bin/activate"
          python -m pip install --upgrade pip wheel
          pip install -r requirements.txt
        '''
      }
    }

    stage('Unit Tests (pytest)') {
      steps {
        sh '''
          set -euxo pipefail
          . "$VENV_DIR/bin/activate"
          pytest -q
        '''
      }
    }

    stage('Build (artifact)') {
      steps {
        sh '''
          set -euxo pipefail
          rm -rf dist
          mkdir -p dist

          tar -czf dist/flask_app.tar.gz \
            . \
            --exclude="./$VENV_DIR" \
            --exclude="./dist" \
            --exclude="./__pycache__" \
            --exclude="./.pytest_cache" \
            --exclude="./.git"

          ls -lah dist
        '''
      }
      post {
        success {
          archiveArtifacts artifacts: 'dist/*.tar.gz', fingerprint: true
        }
      }
    }

    stage('Deploy (local macOS demo)') {
      steps {
        sh '''
          set -euxo pipefail
          . "$VENV_DIR/bin/activate"

          # stop old process on port (simple demo)
          if lsof -iTCP:$APP_PORT -sTCP:LISTEN -t >/dev/null 2>&1; then
            kill -9 $(lsof -iTCP:$APP_PORT -sTCP:LISTEN -t) || true
          fi

          # start gunicorn (expects wsgi.py with: app = create_app() or app = Flask(...))
          nohup gunicorn -w 2 -b 127.0.0.1:$APP_PORT wsgi:app > gunicorn.log 2>&1 &
          sleep 1

          # health check
          curl -s http://127.0.0.1:$APP_PORT/ | cat
          echo "Deployed on http://127.0.0.1:$APP_PORT"
        '''
      }
    }
  }

  post {
    always {
      sh '''
        set +e
        echo "---- gunicorn.log (last 200 lines) ----"
        tail -n 200 gunicorn.log 2>/dev/null || true
      '''
      cleanWs()
    }
  }
}
