// =============================================================================
// Option 1: Declarative Pipeline (Requires: Kubernetes Plugin installed & Jenkins restarted)
// =============================================================================
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    some-label: jenkins-agent-test
spec:
  containers:
  - name: test-runner
    image: alpine:latest
    command:
    - cat
    tty: true
'''
        }
    }
    stages {
        stage('Verify Kubernetes Dynamic Agent') {
            steps {
                container('test-runner') {
                    sh '''
                        echo "=== Verifying Kubernetes Agent Connection ==="
                        uname -a
                        echo "Hostname: $(hostname)"
                        echo "Date: $(date)"
                        echo "Connection to Kubernetes cluster successfully established and verified!"
                    '''
                }
            }
        }
    }
}

/*
// =============================================================================
// Option 2: Scripted Pipeline (Alternative if Declarative Agent isn't registered)
// =============================================================================
podTemplate(yaml: '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: test-runner
    image: alpine:latest
    command:
    - cat
    tty: true
''') {
    node(POD_LABEL) {
        stage('Verify Kubernetes Dynamic Agent') {
            container('test-runner') {
                sh '''
                    echo "=== Verifying Kubernetes Agent Connection ==="
                    uname -a
                    echo "Hostname: $(hostname)"
                '''
            }
        }
    }
}

// =============================================================================
// Option 3: Label-based Agent (If Pod Template is configured in Jenkins Cloud UI)
// =============================================================================
pipeline {
    agent {
        label 'k8s-agent' // Matches label defined in Manage Jenkins > Clouds > Pod Templates
    }
    stages {
        stage('Verify') {
            steps {
                sh 'uname -a'
            }
        }
    }
}
*/
