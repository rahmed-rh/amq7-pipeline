openshift.withCluster() {
 env.NAMESPACE = openshift.project()
 env.APP_ALREADY_EXISTS = true;
 //env.TOKEN = readFile('/var/run/secrets/kubernetes.io/serviceaccount/token').trim()
}

pipeline {
 environment {
  def timestamp = "${System.currentTimeMillis()}"
 }
 options {
  // set a timeout of 20 minutes for this pipeline
  timeout(time: 20, unit: 'MINUTES')
  // when running Jenkinsfile from SCM using jenkinsfilepath the node implicitly does a checkout
  skipDefaultCheckout()
  // Disallow concurrent executions of the Pipeline
  disableConcurrentBuilds()
 }
 agent any
 parameters {
  string(name: 'APP_NAME', defaultValue: 'amq-broker', description: "Application Name - all resources use this name as a label use lowercase")
  string(name: 'NO_OF_REPLICAS', defaultValue: '3', description: "AMQ Cluster Replica size")
 }
 stages {
  stage('initialise') {
   steps {
    echo "NAMESPACE is: ${env.NAMESPACE}"
    echo "Build Number is: ${env.BUILD_NUMBER}"
    echo "Job Name is: ${env.JOB_NAME}"
    sh "oc version"
    sh 'printenv'
   }
  }

  stage('Create ImageStreams & Template') {
   steps {
    script {
     openshift.withCluster() {
      def AMQ_IMAGE_STREAM = "https://raw.githubusercontent.com/jboss-container-images/jboss-amq-7-broker-openshift-image/72-1.3.GA/amq-broker-7-image-streams.yaml"
      def AMQ_TEMPLATE = "https://raw.githubusercontent.com/jboss-container-images/jboss-amq-7-broker-openshift-image/72-1.3.GA/templates/amq-broker-72-persistence-clustered-ssl.yaml"
      echo "Creating/updating AMQ ImageStream & Template"
      openshift.replace("--force", "-f ", "${AMQ_IMAGE_STREAM}")
      openshift.replace("--force", "-f ", "${AMQ_TEMPLATE}")
     }
    }
   }
  }

  stage('Create Needed API Objects') {
   steps {
    script {
     openshift.withCluster() {
      //openshift.verbose() // set logging level for subsequent operations executed (loglevel=8)
      openshift.withProject("${env.NAMESPACE}") {
       if (!openshift.selector('sa', 'broker-service-account').exists()) {
        def sa = ["kind": "ServiceAccount", "apiVersion": "v1", "metadata": ["labels": ["application": "${params.APP_NAME}"], "name": "broker-service-account"]]
        roleObject = openshift.create(sa).object()
       }
       if (!openshift.selector('rolebinding.rbac', 'broker-view-rolebinding').exists()) {
        def roleBinding = ["apiVersion": "rbac.authorization.k8s.io/v1", "kind": "RoleBinding", "metadata": ["labels": ["application": "${params.APP_NAME}"], "name": "broker-view-rolebinding", "namespace": "${env.NAMESPACE}"], "roleRef": ["apiGroup": "rbac.authorization.k8s.io", "kind": "ClusterRole", "name": "view"], "subjects": [
         ["kind": "ServiceAccount", "name": "broker-service-account"]
        ]]
        roleBindingObject = openshift.create(roleBinding).object()
       }
       if (!openshift.selector('secrets', 'amq-app-secret').exists()) {
        // def amqSecretFile = readJSON file: "${workspace}@script/amq-app-secret.json"
        def amqSecretFile = readFile("${workspace}@script/amq-app-secret.json")
        def amqSecretSelector = openshift.create(amqSecretFile).object()
        amqSecretSelector.metadata.put('labels', ["application": "${params.APP_NAME}"])
        openshift.apply(amqSecretSelector) // Patch the object on the server
       }
       def customAMQ7BcSelector = openshift.selector('bc', 'amq7-custom')
       if (customAMQ7BcSelector.exists()) {
        customAMQ7BcSelector.delete()
       }
       def customAMQ7Build = openshift.newBuild("registry.access.redhat.com/amq-broker-7/amq-broker-72-openshift:1.3~https://github.com/rahmed-rh/amq7.git", '--name=amq7-custom').narrow("bc").related('builds')
       timeout(5) {
        customAMQ7Build.watch {
         // Within the body, the variable 'it' is bound to the watched Selector (i.e. builds)
         echo "waiting for ${customAMQ7Build.name()} to complete"

         def allDone = true
         it.withEach {
          if (it.object().status.phase != "Complete") {
           allDone = false
          }
         }
         return allDone;
        }
        openshift.tag("${env.NAMESPACE}/amq7-custom:latest", "${env.NAMESPACE}/amq7-custom:1.${env.BUILD_NUMBER}")
       }
       if (!openshift.selector('sts', "${APP_NAME}-amq").exists()) {
        env.APP_ALREADY_EXISTS = false;
       }
      }
     }
    }
   }
  }

  stage('Create APPLICATION through template') {
   when {
    environment name: 'APP_ALREADY_EXISTS', value: 'false'
   }
   steps {
    script {
     openshift.withCluster() {
      //openshift.verbose() // set logging level for subsequent operations executed (loglevel=8)
      def no_of_replicas = Integer.parseInt("${params.NO_OF_REPLICAS}")
      def amqSts = openshift.newApp("amq-broker-72-persistence-clustered-ssl", "-p APPLICATION_NAME=${params.APP_NAME}", "-p AMQ_QUEUES=demoQueue", "-p AMQ_ADDRESSES=demoTopic", "-p AMQ_USER=amq-demo-user", "-p AMQ_PASSWORD=passw0rd", "-p AMQ_ROLE=amq", "-p AMQ_SECRET=amq-app-secret", "-p AMQ_DATA_DIR=/opt/amq/data", "-p AMQ_DATA_DIR_LOGGING=true", "-p IMAGE=${env.NAMESPACE}/amq7-custom:1.${env.BUILD_NUMBER}", "-p AMQ_PROTOCOL=openwire,amqp,stomp,mqtt,hornetq", "-p VOLUME_CAPACITY=200Mi", "-p AMQ_TRUSTSTORE=amq-broker.jks", "-p AMQ_KEYSTORE=amq-broker.jks", "-p AMQ_TRUSTSTORE_PASSWORD=passw0rd", "-p AMQ_KEYSTORE_PASSWORD=passw0rd", "-p AMQ_CLUSTERED=true", "-p AMQ_REPLICAS=${no_of_replicas}")
      amqSts = amqSts.narrow('statefulset')
      timeout(15) {
       amqSts.watch {
        echo "Waiting for ${it.name()} to be ready"
        return it.object().status.readyReplicas == no_of_replicas
       }
      }

     }
    }
   }
  }

  stage('Recreate PODs to take latest image') {
   when {
    environment name: 'APP_ALREADY_EXISTS', value: 'true'
   }
   steps {
    script {
     openshift.withCluster() {
      //openshift.verbose() // set logging level for subsequent operations executed (loglevel=8)
      def podsSelector = openshift.selector('po', [app: "${APP_NAME}-amq"])
      podsSelector.withEach {
       def podName = "${it.name()}"
       echo "Pod: ${podName} will be deleted"
       it.delete()
       def currentPodsSelector = openshift.selector("${podName}")
       currentPodsSelector.watch {
        echo "Waiting for Pod ${podName} to recreate"
        return it.object().status.containerStatuses[0].ready == true
       }
      }
     }
    }
   }
  }
 }
}
