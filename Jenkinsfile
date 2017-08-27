/**
 * This pipeline will deploy to Kubernetes
 * Using https://github.com/jenkinsci/kubernetes-pipeline-plugin
 *
 * kubernetesApply fails in a cluster with RBAC due to https://github.com/fabric8io/kubernetes-client/issues/850
 */

podTemplate(label: 'deploy', serviceAccount: 'deployer', 
    containers: [containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl', ttyEnabled: true, command: 'cat')]
  ) {

  stage('deployment') {
    node('deploy') {
      checkout scm
      //git 'https://github.com/jenkinsci/kubernetes-plugin.git'
      container('kubectl') {
        sh "kubectl apply -f kubernetes-service.yaml"
        sh "kubectl apply -f kubernetes-v1.yaml"
      }
      // kubernetesApply(file: readFile('kubernetes-service.yaml'), environment: 'kubernetes-plugin')
      // kubernetesApply(file: readFile('kubernetes-v1.yaml'), environment: 'kubernetes-plugin')
    }
  }

  stage('upgrade') {
    timeout(time:1, unit:'DAYS') {
      input message:'Approve upgrade?'
    }
    node('deploy') {
      checkout scm
      //git 'https://github.com/jenkinsci/kubernetes-plugin.git'
      container('kubectl') {
        sh "kubectl apply -f kubernetes-v2.yaml"
      }
      // kubernetesApply(file: readFile('kubernetes-v2.yaml'), environment: 'kubernetes-plugin')
    }
  }
}
