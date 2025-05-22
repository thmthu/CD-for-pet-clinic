pipeline {
  agent any

  parameters {
    string(name: 'SERVICE_NAME', defaultValue: '', description: 'Tên service muốn deploy, ví dụ vets-service, customers-service, visits-service')
    string(name: 'DEPLOY_TAG', defaultValue: 'main', description: 'Tag muốn deploy (newest để lấy tag mới nhất trên DockerHub)')
    string(name: "BRANCH_BUILD", defaultValue: "main", description: "Branch muốn build")

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
          def serviceInput = params.SERVICE_NAME.trim()
          def deployTagInput = params.DEPLOY_TAG.trim()
          def branchBuildInput = params.BRANCH_BUILD.trim()


          // if (!services.contains(serviceInput)) {
          //   error "SERVICE_NAME không hợp lệ. Phải là 1 trong: ${services}"
          // }

          // Lấy tag mới nhất nếu developer chọn 'newest'
          dir("k8s_deploy") {
            sh "git fetch origin"
            sh "git checkout ${branch}"
            def commit = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
              echo "Latest commit on ${branch} is: ${commit}"

              env.LATEST_COMMIT = commit
          }
          def tagToDeploy = deployTagInput
          if (deployTagInput == 'newest') {
            tagToDeploy = '7c0f0d7'
          }

          echo "Deploying service '${serviceInput}' với tag: ${tagToDeploy}"

          // Tạo nội dung override.yaml
          def overrideYaml = ""

          services.each { svc ->
            if (svc == serviceInput) {
              if (tagToDeploy == 'main') {
                // Nếu tag là main thì tắt deploy (hoặc bạn có thể enable mà dùng tag main)
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
    tag: ${tagToDeploy}
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

          writeFile file: "spring-pet-clinic/values.override.yaml", text: overrideYaml.trim()
          echo "Generated values.override.yaml:\n${overrideYaml}"
        }
      }
    }

    stage('Deploy with Helm') {
      steps {
        sh """
          helm upgrade --install petclinic spring-pet-clinic -f spring-pet-clinic/values.yaml
        """
      }
    }
  }
}

// // Hàm lấy tag mới nhất từ DockerHub
// def getLatestTag(serviceName) {
//   def imageName = "${env.IMAGE_PREFIX}-${serviceName}"
//   def tagsJson = sh(
//     script: "curl -s https://hub.docker.com/v2/repositories/${env.DOCKERHUB_USER}/${imageName}/tags?page_size=100",
//     returnStdout: true
//   ).trim()

//   def results = new groovy.json.JsonSlurperClassic().parseText(tagsJson).results
//   def sorted = results.sort { a, b -> b.last_updated <=> a.last_updated }
//   return sorted ? sorted[0].name : "latest"
// }
