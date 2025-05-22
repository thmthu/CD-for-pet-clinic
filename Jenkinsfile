pipeline {
  agent any

  parameters {
    string(name: 'SERVICE_NAME', defaultValue: '', description: 'Tên service muốn deploy, ví dụ vets-service, customers-service, visits-service')
    string(name: 'DEPLOY_TAG', defaultValue: 'main', description: 'Tag muốn deploy (newest để lấy tag mới nhất trên DockerHub)')
    string(name: "BRANCH_BUILD_FOR_VET", defaultValue: "main", description: "Branch muốn build")
    string(name: "BRANCH_BUILD_FOR_CUSTOMER", defaultValue: "main", description: "Branch muốn build")
    string(name: "BRANCH_BUILD_FOR_VISIT", defaultValue: "main", description: "Branch muốn build")
    string 

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
          def branchBuildInput = params.BRANCH_BUILD.trim()
          def serviceBranchMap = [:]
          // for (int i = 0; i < services.size(); i++) {
          //     serviceBranchMap[services[i]] = branchs[i]
          // }

          // In ra để kiểm tra
          serviceBranchMap.each { service, branch ->
              echo "Service: ${service} => Branch: ${branch}"
          }

          // if (!services.contains(serviceInput)) {
          //   error "SERVICE_NAME không hợp lệ. Phải là 1 trong: ${services}"
          // }

          // Lấy tag mới nhất nếu developer chọn 'newest'
          // sh "git clone https://github.com/thmthu/CD-for-pet-clinic.git"
          // if (branchs.contains(branchBuildInput)) {
          //   branchBuildInput = branchBuildInput
          // } else {
          //   branchBuildInput = 'main'
          // }
          dir("CD-for-pet-clinic") {
          echo "Đang ở trong thư mục CD-for-pet-clinic"
          def shortCommits = []
          branchBuilds.each { branchName ->
            if(branch == 'main') {
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
          // if (branchBuildInput == 'main') {
          //   tagToDeploy = 'latest'
          //   echo "Deploying from main branch, using tag: ${tagToDeploy}"
          // }else {
          //   tagToDeploy = "${env.LATEST_COMMIT}"
          //   echo "Deploying from ${branchBuildInput} branch, using tag: ${tagToDeploy}"
          // }

          echo "Deploying service '${serviceInput}' với tag: ${tagToDeploy}"

          // Tạo nội dung override.yaml
          def overrideYaml = ""

          services.each { svc ->
            if (svc == serviceInput) {
              if (tagToDeploy == serviceBranchMap[svc]) {
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
