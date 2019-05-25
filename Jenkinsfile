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

		stage('Create Needed API Objects') {
			steps {
				script {
					openshift.withCluster() {
						//openshift.verbose() // set logging level for subsequent operations executed (loglevel=8)
						openshift.withProject("${env.NAMESPACE}") {
              dummy=openshift.selector('sa', 'broker-service-account')
              echo "${dummy}"
              echo "${dummy.exists()}"
							if (!openshift.selector('sa', 'broker-service-account').exists()) {
								def sa = ["kind": "ServiceAccount", "apiVersion": "v1", "metadata": ["labels": ["app": "${APP_NAME}"], "name": "broker-service-account"]]
								roleObject = openshift.create(role).object()
							}
							if (!openshift.selector('rolebinding.rbac', 'broker-role-binding').exists()) {
								def roleBinding = ["apiVersion": "rbac.authorization.k8s.io/v1", "kind": "RoleBinding", "metadata": ["labels": ["app": "${APP_NAME}"], "name": "broker-role-binding", "namespace": "${env.NAMESPACE}"], "roleRef": ["apiGroup": "rbac.authorization.k8s.io", "kind": "ClusterRole", "name": "view"], "subjects": [["kind": "ServiceAccount", "name": "broker-service-account"]]]
								roleBindingObject = openshift.create(roleBinding).object()
							}
							if (!openshift.selector('secrets', 'amq-app-secret').exists()) {
								def amqSecret = readFile("amq-app-secret.json")
                echo "${amqSecret}"
								amqSecret.Object()
								amqSecret.metadata.labels['app'] = "${APP_NAME}"
								openshift.create(amqSecret)
							}
						}
					}
				}
			}
		}
	}
}
