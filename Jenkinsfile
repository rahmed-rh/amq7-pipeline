openshift.withCluster() {
	def PROJECT_NAME="amq-s2i-raif"
	stage('Preparing AMQ Env') {
    	def sa = [
		  "kind": "ServiceAccount",
		  "apiVersion": "v1",
		  "metadata": [
		   "name": "broker-service-account"
		  ]
		]
    	saObject = openshift.create(sa).object()

		def roleBinding = [
			apiVersion: "rbac.authorization.k8s.io/v1",
			kind: "RoleBinding",
			metadata: [
  				name: "broker-role-binding",
  				namespace: ${PROJECT_NAME}
			],
			roleRef: [
  				apiGroup: "rbac.authorization.k8s.io",
  				kind: "Role",
  				name: "view"
			],
			subjects: [
				[
					kind: "ServiceAccount",
					name: "broker-service-account"
				]
			]
		]
		roleBindingObject = openshift.create(roleBinding).object()
		
		openshift.create( secret, 'generic', 'amq-app-secret', '--from-file=amq-broker.jks' )
		openshift.newBuild("registry.access.redhat.com/amq-broker-7/amq-broker-72-openshift:1.3","https://github.com/rahmed-rh/amq7.git")
	}
}
