pipeline {
  agent any

  parameters {
    string(name: 'SERVICE_NAME', defaultValue: '', description: 'Tên service muốn deploy, ví dụ vets-service, customers-service, visits-service')
    string(name: 'DEPLOY_TAG', defaultValue: 'main', description: 'Tag muốn deploy (newest để lấy tag mới nhất trên DockerHub)')
    string(name: "BRANCH_BUILD_FOR_VET", defaultValue: "main", description: "Branch muốn build")
    string(name: "BRANCH_BUILD_FOR_CUSTOMER", defaultValue: "main", description: "Branch muốn build")
    string(name: "BRANCH_BUILD_FOR_VISIT", defaultValue: "main", description: "Branch muốn build")
  }

  environment {
    DOCKERHUB_USER = 'mytruong28022004'
    IMAGE_PREFIX = 'spring-petclinic'
  }
  
  stages {
    stage('Get Branches') {
      steps {
        script {
          def branches = sh(script: "git ls-remote --heads https://github.com/thmthu/CD-for-pet-clinic.git | awk '{print \$2}' | sed 's|refs/heads/||'", returnStdout: true)
                      .trim()
                      .split("\n")
          echo "Branches: ${branches}"
          branches.each { branch ->
            echo "Found branch: ${branch}"
          }
          env.BRANCH_LIST = branches.join(',')
        }
      }
    }
    
    stage('Prepare Values Override') {
      steps {
        script {
          def services = ['vets-service', 'customers-service', 'visits-service']
          def branchs = env.BRANCH_LIST.split(',')
          def branchBuilds = [params.BRANCH_BUILD_FOR_VET, params.BRANCH_BUILD_FOR_CUSTOMER, params.BRANCH_BUILD_FOR_VISIT]
          def serviceInput = params.SERVICE_NAME.trim()
          def deployTagInput = params.DEPLOY_TAG.trim()
          def serviceBranchMap = [:]

          serviceBranchMap.each { service, branch ->
            echo "Service: ${service} => Branch: ${branch}"
          }

          dir("CD-for-pet-clinic") {
            echo "Đang ở trong thư mục CD-for-pet-clinic"
            def shortCommits = []
            branchBuilds.each { branchName ->
              if (branch == 'main') {
                shortCommits.add("latest")
              } else {
                sh "git fetch origin"
                sh "git checkout ${branchName}"

                def fullCommit = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                def shortCommit = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()

                shortCommits.add(shortCommit)
              }
            }
            
            echo "short commits: ${shortCommits}"
            for (int i = 0; i < services.size(); i++) {
              serviceBranchMap[services[i]] = shortCommits[i]
            }
            
            def tagToDeploy = deployTagInput  
            echo "Deploying service '${serviceInput}' với tag: ${tagToDeploy}"

            // Tạo nội dung override.yaml
            def overrideYaml = ""

            services.each { svc ->
              if (svc == serviceInput) {
                if (tagToDeploy == serviceBranchMap[svc]) {
                  // Nếu tag là main thì tắt deploy
                  overrideYaml += """
${svc}:
  enabled: false
"""
                } else {
                  // Enable và set tag cho service muốn deploy
                  overrideYaml += """
${svc}:
  enabled: true
  image:
    repository: ${DOCKERHUB_USER}/${IMAGE_PREFIX}-${svc}
    tag: ${serviceBranchMap[svc]}
"""
                }
              } else {
                // Các service khác disable
                overrideYaml += """
${svc}:
  enabled: false
"""
              }
            }

            writeFile file: "spring-pet-clinic/values_devCD.override.yaml", text: overrideYaml.trim()
            echo "Generated values_devCD.override.yaml:\n${overrideYaml}"
          }
        }
      }
    }

    stage('Deploy with Helm') {
      steps {
        sh """
          helm upgrade --install petclinic spring-pet-clinic -f spring-pet-clinic/values_devCD.yaml -n ${shortCommit} --create-namespace
        """
      }
    }

    stage('Deployment Link') {
      steps {
        script {
          def link = "spring-pet-clinic.local"

          echo "🟢 [View Deployed Service](${link})"
          currentBuild.description = "<a href='${link}' target='_blank'>${link}</a>"
        }
      }
    }

  }
}