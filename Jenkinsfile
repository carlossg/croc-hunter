#!/usr/bin/groovy

// load pipeline functions
// Requires pipeline-github-lib plugin to load library from github
@Library('github.com/carlossg/helm-jenkins-pipeline@master')
import groovy.json.*

def pipeline = new io.estrado.Pipeline()

timestamps {

podTemplate(name:'croc-hunter', label: 'jenkins-pipeline', containers: [
    containerTemplate(name: 'jnlp', image: 'jenkins/jnlp-slave:3.10-1-alpine', args: '${computer.jnlpmac} ${computer.name}', workingDir: '/home/jenkins', resourceRequestCpu: '200m', resourceLimitCpu: '200m', resourceRequestMemory: '256Mi', resourceLimitMemory: '256Mi'),
    containerTemplate(name: 'docker', image: 'csanchez/google-cloud-sdk-docker-client', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'golang', image: 'golang:1.8.3', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'helm', image: 'lachlanevenson/k8s-helm:v2.5.0', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.7.8', command: 'cat', ttyEnabled: true)
],
serviceAccount: 'deployer',
volumes:[
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
]){

  node ('jenkins-pipeline') {

    def pwd = pwd()
    def chart_dir = "${pwd}/charts/croc-hunter"

    checkout scm

    // read in required jenkins workflow config values
    def inputFile = readFile('Jenkinsfile.json')
    def config = new groovy.json.JsonSlurperClassic().parseText(inputFile)
    println "pipeline config ==> ${config}"

    // continue only if pipeline enabled
    if (!config.pipeline.enabled) {
        println "pipeline disabled"
        return
    }

    // set additional git envvars for image tagging
    pipeline.gitEnvVars()

    def namespace = "croc-hunter"

    // If pipeline debugging enabled
    if (config.pipeline.debug) {
      println "DEBUG ENABLED"
      sh "env | sort"

      println "Runing kubectl/helm tests"
      container('kubectl') {
        pipeline.kubectlTest()
      }
      container('helm') {
        pipeline.helmConfig()
      }
    }

    def acct = pipeline.getContainerRepoAcct(config)

    // tag image with version, and branch-commit_id
    def image_tags_map = pipeline.getContainerTags(config)

    // compile tag list
    def image_tags_list = pipeline.getMapValues(image_tags_map)

    stage ('compile and test') {

      container('golang') {
        sh "go test -v -race ./..."
        sh "make bootstrap build"
      }
    }

    // stage ('test deployment') {
    //
    //   container('helm') {
    //
    //     // run helm chart linter
    //     pipeline.helmLint(chart_dir)
    //
    //     // run dry-run helm chart installation
    //     pipeline.helmDeploy(
    //       dry_run       : true,
    //       name          : config.app.name,
    //       namespace     : config.app.name,
    //       version_tag   : image_tags_list.get(0),
    //       chart_dir     : chart_dir,
    //       replicas      : config.app.replicas,
    //       cpu           : config.app.cpu,
    //       memory        : config.app.memory,
    //       hostname      : config.app.hostname
    //     )
    //
    //   }
    // }

    stage ('publish container') {

      container('docker') {

        // perform docker login to quay as the docker-pipeline-plugin doesn't work with the next auth json format
        // withCredentials([[$class          : 'UsernamePasswordMultiBinding', credentialsId: config.container_repo.jenkins_creds_id,
        //                 usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
        //   sh "docker login -e ${config.container_repo.dockeremail} -u ${env.USERNAME} -p ${env.PASSWORD} quay.io"
        // }


        // build and publish container
        // pipeline.containerBuildPub(
        //     dockerfile: config.container_repo.dockerfile,
        //     host      : config.container_repo.host,
        //     acct      : acct,
        //     repo      : config.container_repo.repo,
        //     tags      : image_tags_list,
        //     auth_id   : config.container_repo.jenkins_creds_id
        // )

        sh """
        docker build --build-arg VCS_REF=${env.GIT_SHA} --build-arg BUILD_DATE=`date -u +'%Y-%m-%dT%H:%M:%SZ'` -t ${config.container_repo.host}/${config.container_repo.master_acct}/${config.container_repo.repo} .
        gcloud docker -- push ${config.container_repo.host}/${config.container_repo.master_acct}/${config.container_repo.repo}
        """
      }

    }

    if (env.BRANCH_NAME =~ "demo" ) {
      stage ('deploy to k8s') {

        container('kubectl') {
          //sh "kubectl create namespace ${namespace}"
          sh "kubectl apply -n ${namespace} -f kubernetes.yaml"
        }

        // run tests
        sh "wget http://croc-hunter.croc-hunter"

        // delete test deployment
        container('kubectl') {
          sh "kubectl delete -n ${namespace} -f kubernetes.yaml"
        }


        // container('helm') {
        //   // Deploy using Helm chart
        //   pipeline.helmDeploy(
        //     dry_run       : false,
        //     name          : env.BRANCH_NAME.toLowerCase(),
        //     namespace     : env.BRANCH_NAME.toLowerCase(),
        //     version_tag   : image_tags_list.get(0),
        //     chart_dir     : chart_dir,
        //     replicas      : config.app.replicas,
        //     cpu           : config.app.cpu,
        //     memory        : config.app.memory,
        //     hostname      : config.app.hostname
        //   )
        //
        //   //  Run helm tests
        //   if (config.app.test) {
        //     pipeline.helmTest(
        //       name        : env.BRANCH_NAME.toLowerCase()
        //     )
        //   }
        //
        //   // delete test deployment
        //   pipeline.helmDelete(
        //       name       : env.BRANCH_NAME.toLowerCase()
        //   )
        // }
      }
    }
  }

  // deploy only the master branch
  if (env.BRANCH_NAME == 'demo') {
    timeout(time:1, unit:'DAYS') {
      input id: 'approve', message:'Approve deployment?'
    }

    node ('jenkins-pipeline') {

      stage ('deploy to k8s') {

        container('kubectl') {
          // sh "kubectl create namespace ${namespace}"
          sh "kubectl apply -n ${namespace} -f kubernetes.yaml"
        }

        // container('helm') {
        //   // Deploy using Helm chart
        //   pipeline.helmDeploy(
        //     dry_run       : false,
        //     name          : config.app.name,
        //     namespace     : config.app.name,
        //     version_tag   : image_tags_list.get(0),
        //     chart_dir     : chart_dir,
        //     replicas      : config.app.replicas,
        //     cpu           : config.app.cpu,
        //     memory        : config.app.memory,
        //     hostname      : config.app.hostname
        //   )
        //
        //   //  Run helm tests
        //   if (config.app.test) {
        //     pipeline.helmTest(
        //       name          : config.app.name
        //     )
        //   }
        // }
      }
    }
  }
}

}
