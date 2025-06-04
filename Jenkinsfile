pipeline {
  agent any

  parameters {
    string(name: "BRANCH_BUILD_FOR_VET", defaultValue: "main", description: "Branch muốn build")
    string(name: "BRANCH_BUILD_FOR_CUSTOMER", defaultValue: "main", description: "Branch muốn build")
    string(name: "BRANCH_BUILD_FOR_VISIT", defaultValue: "main", description: "Branch muốn build")
    string(name: 'RELEASE_NAME', defaultValue: 'dev', description: 'Tên release muốn deploy')
  }

  environment {
    DOCKERHUB_USER = 'mytruong28022004'
    IMAGE_PREFIX = 'spring-petclinic'
    KUBECONFIG = "/etc/rancher/k3s/k3s.yaml"
    JENKINS_URL = "http://35.209.75.248:8080/"
    VALUES_FILE = "values_devCD.yaml"

    // Observability
    OBS_NAMESPACE = "monitoring"
    OBS_RELEASE = "observability"
    OBS_VALUES_FILE = "observability/values_devCD.yaml"
  }

  stages {
    stage('Get Branches') {
      steps {
        script {
          def branches = sh(script: "git ls-remote --heads https://github.com/thmthu/CD-for-pet-clinic.git | awk '{print \$2}' | sed 's|refs/heads/||'", returnStdout: true).trim().split("\n")
          env.BRANCH_LIST = branches.join(',')
        }
      }
    }

    stage('Prepare Values Override') {
      steps {
        script {
          def services = ['vets-service', 'customers-service', 'visits-service']
          def branchBuilds = [params.BRANCH_BUILD_FOR_VET, params.BRANCH_BUILD_FOR_CUSTOMER, params.BRANCH_BUILD_FOR_VISIT]
          def serviceBranchMap = [:]

          sh 'rm -rf spring-petclinic-microservices-fork'
          sh "git clone https://github.com/MyTruong28022004/spring-petclinic-microservices-fork.git"
          dir("spring-petclinic-microservices-fork") {
            def shortCommits = []
            branchBuilds.each { branchName ->
              if (branchName == 'main') {
                shortCommits.add("latest")
              } else {
                sh "git fetch origin"
                sh "git checkout ${branchName}"
                def shortCommit = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()
                shortCommits.add(shortCommit)
              }
            }

            for (int i = 0; i < services.size(); i++) {
              serviceBranchMap[services[i]] = shortCommits[i]
              if(shortCommits[i] != "latest"){
                env.COMMIT = shortCommits[i]
              }
            }

            def overrideYaml = ""
            services.each { svc ->
              overrideYaml += """
              ${svc}:
                enabled: true
                image:
                  repository: ${DOCKERHUB_USER}/${IMAGE_PREFIX}-${svc}
                  tag: ${serviceBranchMap[svc]}
              """
            }

            writeFile file: "spring-pet-clinic/values_devCD.override.yaml", text: overrideYaml.trim()
          }
        }
      }
    }

    stage('Update Hosts in values.yaml') {
      steps {
        script {
          if(!env.COMMIT){ env.COMMIT = "main" }
          env.gatewayHost = "spring-pet-clinic-dev-${env.COMMIT}.local"
          env.adminHost = "spring-pet-clinic-dev-${env.COMMIT}-server.local"

          sh """
            sed -i 's|host: spring-pet-clinic\\.local|host: ${env.gatewayHost}|' spring-pet-clinic/${VALUES_FILE}
            sed -i 's|host: spring-pet-clinic-admin\\.local|host: ${env.adminHost}|' spring-pet-clinic/${VALUES_FILE}
          """
        }
      }
    }

    stage('Deploy PetClinic with Helm') {
      steps {
        script {
          echo "🚀 Deploying PetClinic services..."
          sh """
            helm upgrade --install ${params.RELEASE_NAME} spring-pet-clinic \
              -f spring-pet-clinic/values_devCD.yaml \
              -f spring-pet-clinic/values_devCD.override.yaml \
              -n dev-${env.COMMIT} --create-namespace
          """
        }
      }
    }

    stage('Deploy Observability Stack') {
      steps {
        script {
          echo "🚀 Deploying Observability Stack (Prometheus, Grafana, Loki, Zipkin)..."
          sh """
            helm upgrade --install ${OBS_RELEASE} ./observability \
              -f ${OBS_VALUES_FILE} \
              -n ${OBS_NAMESPACE} --create-namespace
          """
        }
      }
    }

    stage('Deployment Links') {
      steps {
        script {
          def deleteJobUrl = "${env.JENKINS_URL}/job/delete-deployment/buildWithParameters?token=delete-token&RELEASE_NAME=${params.RELEASE_NAME}&NAMESPACE=dev-${env.COMMIT}"
          def grafanaUrl = "http://grafana.${OBS_NAMESPACE}.svc.cluster.local"
          def prometheusUrl = "http://prometheus.${OBS_NAMESPACE}.svc.cluster.local:9090"
          def zipkinUrl = "http://zipkin.${OBS_NAMESPACE}.svc.cluster.local:9411"
          def lokiUrl = "http://loki.${OBS_NAMESPACE}.svc.cluster.local:3100"

          currentBuild.description = """
            <a href='http://${env.gatewayHost}' target='_blank'>App Gateway</a><br>
            <a href='http://${env.adminHost}' target='_blank'>Admin Interface</a><br>
            <a href='${grafanaUrl}' target='_blank'>Grafana</a><br>
            <a href='${prometheusUrl}' target='_blank'>Prometheus</a><br>
            <a href='${lokiUrl}' target='_blank'>Loki</a><br>
            <a href='${zipkinUrl}' target='_blank'>Zipkin</a><br>
            <a href='${deleteJobUrl}'>[🗑 Xóa release ${params.RELEASE_NAME}]</a>
          """
        }
      }
    }
  }
}