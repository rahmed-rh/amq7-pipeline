openshift.withCluster() {
	env.NAMESPACE = openshift.project()
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
		string(name: 'APP_NAME', defaultValue: 'AMQ-BROKER', description: "Application Name - all resources use this name as a label")
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
								def sa = ["kind": "ServiceAccount", "apiVersion": "v1", "metadata": ["labels": ["app": "${APP_NAME}"], "name": "broker-service-account"]]
								roleObject = openshift.create(sa).object()
							}
							if (!openshift.selector('rolebinding.rbac', 'broker-role-binding').exists()) {
								def roleBinding = ["apiVersion": "rbac.authorization.k8s.io/v1", "kind": "RoleBinding", "metadata": ["labels": ["app": "${APP_NAME}"], "name": "broker-role-binding", "namespace": "${env.NAMESPACE}"], "roleRef": ["apiGroup": "rbac.authorization.k8s.io", "kind": "ClusterRole", "name": "view"], "subjects": [["kind": "ServiceAccount", "name": "broker-service-account"]]]
								roleBindingObject = openshift.create(roleBinding).object()
							}
							if (!openshift.selector('secrets', 'amq-app-secret').exists()) {
                // def amqSecretFile = readJSON file: "${workspace}@script/amq-app-secret.json"
								def amqSecretFile = readFile("${workspace}@script/amq-app-secret.json")
                def amqSecretSelector = openshift.create( amqSecretFile ).object()
                amqSecretSelector.metadata.put('labels', ["app": "${APP_NAME}"])
                openshift.apply(amqSecretSelector) // Patch the object on the server

							}
              def customAMQ7BcSelector=openshift.selector('bc', 'amq7-custom')
              if (customAMQ7BcSelector.exists()) {
                customAMQ7BcSelector.delete()
              }
              def customAMQ7Build = openshift.newBuild("registry.access.redhat.com/amq-broker-7/amq-broker-72-openshift:1.3~https://github.com/rahmed-rh/amq7.git",'--name=amq7-custom','--labels="app"="${APP_NAME}"')
              timeout(2) {
                     customAMQ7Build.watch {
                         // Within the body, the variable 'it' is bound to the watched Selector (i.e. builds)
                         echo "So far, ${customAMQ7Build.name()} has created build: ${it.names()}"
                         // End the watch only once a build object has been created.
                         return it.count() > 0
                     }
              }


						}
					}
				}
			}
		}


    stage('Recreate PODs to take latest image') {
			steps {
				script {
					openshift.withCluster() {
						//openshift.verbose() // set logging level for subsequent operations executed (loglevel=8)

					}
				}
			}
		}


	}
}
