pipeline {
    agent any
    environment {
        VENV_DIR = 'venv'
        // Prevents Jenkins from automatically killing background processes at the end of a stage
        JENKINS_NODE_COOKIE = 'dontKillMe' 
    }
    options {
        timestamps()
        disableConcurrentBuilds()
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Set Up Python Environment') {
            steps {
                // Use a single multi-line string so environment variables and commands persist correctly
                sh """
                python3 -m venv ${VENV_DIR}
                . ${VENV_DIR}/bin/activate
                pip install --upgrade pip
                pip install -r requirements.txt
                pip install requests
                """
            }
        }
        stage('Train Model') {
            steps {
                // Call the python binary inside the venv directly to avoid activation loss
                sh "${VENV_DIR}/bin/python train_model.py"
            }
        }
        stage('Start API & Smoke Test') {
            steps {
                sh """
                # Start API using the venv's python binary
                nohup ${VENV_DIR}/bin/python app.py > app.log 2>&1 &
                echo \$! > app.pid
                
                echo "Waiting for API to become ready..."
                ready=0
                for i in \$(seq 1 30); do
                    if curl -s -o /dev/null http://127.0.0.1:5000/; then
                        ready=1
                        echo "API is up"
                        break
                    fi
                    sleep 1
                done
                
                if [ \$ready -ne 1 ]; then
                    echo "API did not start in time"
                    cat app.log || true
                    exit 1
                fi
                
                # Run test using the venv's python binary
                ${VENV_DIR}/bin/python test_prediction.py
                """
            }
        }
    }
    post {
        always {
            sh """
            if [ -f app.pid ]; then
                kill \$(cat app.pid) 2>/dev/null || true
                rm -f app.pid
            fi
            # Safeguard to free up port 5000
            fuser -k 5000/tcp 2>/dev/null || true
            """
            archiveArtifacts artifacts: 'uptor_model.pkl, app.log', allowEmptyArchive: true
        }
        success {
            echo 'Build, train, and smoke test succeeded.'
        }
        failure {
            echo 'Pipeline failed — check app.log and the console output above for details.'
        }
        cleanup {
            sh "rm -rf ${VENV_DIR}"
        }
    }
}
