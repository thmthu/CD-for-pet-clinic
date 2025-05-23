pipeline {
  agent any

  parameters {
    string(name: 'SERVICE_NAME', defaultValue: '', description: 'Tên service muốn deploy, ví dụ vets-service, customers-service, visits-service')
    string(name: 'DEPLOY_TAG', defaultValue: 'main', description: 'Tag muốn deploy (newest để lấy tag mới nhất trên DockerHub)')
    string(name: "BRANCH_BUILD_FOR_VET", defaultValue: "main", description: "Branch muốn build")
    string(name: "BRANCH_BUILD_FOR_CUSTOMER", defaultValue: "main", description: "Branch muốn build")
    string(name: "BRANCH_BUILD_FOR_VISIT", defaultValue: "main", description: "Branch muốn build")
    string(name: 'RELEASE_NAME', defaultValue: 'dev', description: 'Tên release muốn deploy')
  }

  environment {
    DOCKERHUB_USER = 'mytruong28022004'
    IMAGE_PREFIX = 'spring-petclinic'
    KUBECONFIG = "/etc/rancher/k3s/k3s.yaml"
    JENKINS_URL= "http://35.209.75.248:8080/"
    VALUES_FILE = "values_devCD.yaml"
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
          sh 'rm -rf spring-petclinic-microservices-fork'
          sh "git clone https://github.com/MyTruong28022004/spring-petclinic-microservices-fork.git"
          dir("spring-petclinic-microservices-fork") {
            echo "Đang ở trong thư mục spring-petclinic-microservices-fork"
            def shortCommits = []
            branchBuilds.each { branchName ->
              if (branchName == 'main') {
                shortCommits.add("latest")
              } else {
                sh "git fetch origin"
                sh "git checkout ${branchName}"

                def fullCommit = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                echo "Full commit hash: ${fullCommit}"
                def shortCommit = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()

                shortCommits.add(shortCommit)
              }
            }
            
            echo "short commits: ${shortCommits}"
            for (int i = 0; i < services.size(); i++) {
              serviceBranchMap[services[i]] = shortCommits[i]
              if(commitId != 'latest') {
                env.COMMIT_ID = shortCommits[i]
              }
               
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
  enabled: true
  image:
    repository: ${DOCKERHUB_USER}/${IMAGE_PREFIX}-${svc}
    tag: ${serviceBranchMap[svc]}
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
  enabled: true
  image:
    repository: ${DOCKERHUB_USER}/${IMAGE_PREFIX}-${svc}
    tag: ${serviceBranchMap[svc]}
"""
              }
            }

            writeFile file: "spring-pet-clinic/values_devCD.override.yaml", text: overrideYaml.trim()
            echo "Generated values_devCD.override.yaml:\n${overrideYaml}"
          }
        }
      }
    }
    stage('Update Hosts in values.yaml') {
      steps {
        script {
          sh """
              yq e '
              .gateway.ingress.hosts[0].host = "spring-pet-clinic-dev-${env.COMMIT_ID}.local"
              ' -i ${VALUES_FILE}
              """

                    // Cập nhật admin ingress host
                    sh """
                        yq e '
                          .admin.ingress.hosts[0].host = "spring-pet-clinic-dev-${env.COMMIT_ID}-server.local"
                        ' -i ${VALUES_FILE}
                    """

                    echo "Updated ingress hosts with commit ID ${env.COMMIT_ID}"
                }
            }
        }
    stage('Deploy with Helm') {
      steps {
        script {
          // Get the short Git commit hash
          def shortCommit = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()
          echo "Short commit hash: ${shortCommit}"

          // Run the Helm deployment with the namespace
          sh """
            helm upgrade --install petclinic spring-pet-clinic \
              -f spring-pet-clinic/values_devCD.yaml \
              -n dev --create-namespace
          """
        }
      }
    }
    stage('Deployment Link') {
      steps {
        script {
          def link = "http://spring-pet-clinic.local"

          echo "🟢 [View Deployed Service](${link})"
          currentBuild.description = "[${link}](${link})"
        }
      }
    }
    stage('Delete Link') {
      steps {
        script {
          def link = "http://spring-pet-clinic.local"

          echo "🔴 [Delete Deployed Service](${link})"
          currentBuild.description = "[${link}](${link})"
        }
      }
    }
  }
}