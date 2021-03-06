def ciProject = 'labs-ci-cd'
def testProject = 'labs-test'
def devProject = 'labs-dev'
openshift.withCluster() {
  openshift.withProject() {
    ciProject = openshift.project()
    testProject = ciProject.replaceFirst(/^labs-ci-cd/, 'labs-test')
    devProject = ciProject.replaceFirst(/^labs-ci-cd/, 'labs-dev')
  }
}

def buildConfig = { project, namespace, buildSecret, fromImageStream ->
  if (!fromImageStream) {
    fromImageStream = 'registry.access.redhat.com/rhoar-nodejs/nodejs-10'
  }
  def template = """
---
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  labels:
    build: ${project}
  name: ${project}
  namespace: ${namespace}
spec:
  failedBuildsHistoryLimit: 5
  nodeSelector: null
  output:
    to:
      kind: ImageStreamTag
      name: '${project}:latest'
  postCommit: {}
  resources: {}
  runPolicy: Serial
  source:
    binary: {}
    type: Binary
  strategy:
    sourceStrategy:
      from:
        kind: ImageStreamTag
        name: '${fromImageStream}'
        namespace: openshift
    type: Source
  successfulBuildsHistoryLimit: 5
  triggers:
    - github:
        secret: ${buildSecret}
      type: GitHub
    - generic:
        secret: ${buildSecret}
      type: Generic
"""
  openshift.withCluster() {
    openshift.apply(template, "--namespace=${namespace}")
  }
}

def deploymentConfig = {project, ciNamespace, targetNamespace ->
  def template = """
---
apiVersion: v1
kind: List
items:
- apiVersion: v1
  kind: ImageStream
  metadata:
    labels:
      build: '${project}'
    name: '${project}'
    namespace: '${targetNamespace}'
  spec: {}
- apiVersion: v1
  kind: DeploymentConfig
  metadata:
    labels:
      app: '${project}'
    name: '${project}'
  spec:
    replicas: 1
    selector:
      name: '${project}'
    strategy:
      activeDeadlineSeconds: 21600
      resources: {}
      rollingParams:
        intervalSeconds: 1
        maxSurge: 25%
        maxUnavailable: 25%
        timeoutSeconds: 600
        updatePeriodSeconds: 1
      type: Rolling
    template:
      metadata:
        creationTimestamp: null
        labels:
          name: '${project}'
      spec:
        containers:
          - image: '${project}'
            imagePullPolicy: Always
            name: '${project}'
            env:
            - name: KUBERNETES_NAMESPACE
              value: ${targetNamespace}
            ports:
              - containerPort: 8080
                protocol: TCP
        restartPolicy: Always
        securityContext: {}
        terminationGracePeriodSeconds: 30
    test: false
    triggers:
      - type: ConfigChange
      - imageChangeParams:
          automatic: true
          containerNames:
            - '${project}'
          from:
            kind: ImageStreamTag
            name: '${project}:latest'
        type: ImageChange
- apiVersion: v1
  kind: Service
  metadata:
    labels:
      name: '${project}'
    name: '${project}'
  spec:
    ports:
      - name: http
        port: 8080
        protocol: TCP
        targetPort: 8080
    selector:
      name: '${project}'
    sessionAffinity: None
    type: ClusterIP
- apiVersion: v1
  kind: Route
  metadata:
    labels:
      name: '${project}'
    name: '${project}'
  spec:
    port:
      targetPort: http
    to:
      kind: Service
      name: '${project}'
      weight: 100
    wildcardPolicy: None
"""
  openshift.withCluster() {
    openshift.apply(template, "--namespace=${targetNamespace}")
  }
}

pipeline {
  agent {
    label 'jenkins-slave-npm'
  }
  environment {
    PROJECT_NAME = 'noun-service'
    KUBERNETES_NAMESPACE = "${ciProject}"
  }
stages {
    stage('Compile') {
        steps {
            sh 'npm install'
        }
    }
    stage('OpenShift Deployments') {
      parallel {
        stage('Create Binary BuildConfig') {
          steps {
            script {
              buildConfig(PROJECT_NAME, ciProject, UUID.randomUUID().toString())
            }
          }
        }
        stage('Create Test Deployment') {
          steps {
            script {
              deploymentConfig(PROJECT_NAME, ciProject, testProject)
            }
          }
        }
        stage('Create Dev Deployment') {
          steps {
            script {
              deploymentConfig(PROJECT_NAME, ciProject, devProject)
            }
          }
        }
      }
    }
    stage('Build Image') {
      steps {
        script {
          openshift.selector('bc', PROJECT_NAME).startBuild("--from-dir=./", '--wait')
        }
      }
    }
    stage('Promote to TEST') {
      steps {
        script {
          openshift.tag("${PROJECT_NAME}:latest", "${testProject}/${PROJECT_NAME}:latest")
        }
      }
    }
    stage('Promote to DEMO') {
      input {
        message "Promote service to DEMO environment?"
        ok "PROMOTE"
      }
      steps {
        script {
          openshift.tag("${PROJECT_NAME}:latest", "${devProject}/${PROJECT_NAME}:latest")
        }
      }
    }
  }
}