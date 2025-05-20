pipeline {
  agent any
  parameters {
    string(name: 'vets_branch', defaultValue: 'main')
    string(name: 'customers_branch', defaultValue: 'main')
    string(name: 'visit_branch', defaultValue: 'main')
    // Add more for each service
  }
  environment {
    DOCKERHUB_PREFIX = "yourdockerhub"
  }
  stages {
    stage('Resolve Tags') {
      steps {
        script {
          def resolveTag = { branch ->
            return branch == 'main' ? 'latest' : getCommitId(branch)
          }

          VETS_TAG = resolveTag(params.vets_branch)
          CUSTOMERS_TAG = resolveTag(params.customers_branch)
          VISIT_TAG = resolveTag(params.visit_branch)
        }
      }
    }

    stage('Deploy to K8s') {
      steps {
        script {
          sh """
          
          help upgrade --install <chart_name> <chart_path> --values <values_file> -n <namespace>
          ./deploy/scripts/deploy.sh \\
            --vets ${DOCKERHUB_PREFIX}/vets-service:${VETS_TAG} \\
            --customers ${DOCKERHUB_PREFIX}/customers-service:${CUSTOMERS_TAG} \\
            --visit ${DOCKERHUB_PREFIX}/visit-service:${VISIT_TAG}
          """
        }
      }
    }
  }
}

// Hàm lấy commit ID từ nhánh
def getCommitId(String branchName) {
  def id = sh(returnStdout: true, script: "git ls-remote origin ${branchName} | cut -c1-7").trim()
  return id
}
