pipeline {
  agent any

  parameters {
    string(name: 'SERVICE_NAME', defaultValue: '', description: 'Tên service muốn deploy, ví dụ vets-service, customers-service, visits-service')
    string(name: 'DEPLOY_TAG', defaultValue: 'main', description: 'Tag muốn deploy (newest để lấy tag mới nhất trên DockerHub)')
  }

  environment {
    DOCKERHUB_USER = 'mytruong28022004'
    IMAGE_PREFIX = 'spring-petclinic'
    KUBECONFIG = "/etc/rancher/k3s/k3s.yaml"
  }

  stages {
    stage('Prepare Values Override') {
      steps {
        script {
          def services = ['vets-service', 'customers-service', 'visits-service']
          def serviceInput = params.SERVICE_NAME.trim()
          def deployTagInput = params.DEPLOY_TAG.trim()

          // if (!services.contains(serviceInput)) {
          //   error "SERVICE_NAME không hợp lệ. Phải là 1 trong: ${services}"
          // }

          // Lấy tag mới nhất nếu developer chọn 'newest'
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
          helm upgrade --install petclinic spring-pet-clinic -f spring-pet-clinic/values_dev.yaml -n dev \
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
