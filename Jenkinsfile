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

		def role= [
			apiVersion: "authorization.openshift.io/v1",
			kind: "Role",
			metadata: [
				name: "broker-role",
	  			namespace: "${PROJECT_NAME}"
			],
			"rules": [
				[
				    "apiGroups": [
				        ""
				    ],
				    "attributeRestrictions": null,
				    "resources": [
				        "endpoints"
				    ],
				    "verbs": [
				        "create",
				        "delete",
				        "deletecollection",
				        "get",
				        "list",
				        "patch",
				        "update",
				        "watch"
				    ]
				],
				[
				    "apiGroups": [
				        ""
				    ],
				    "attributeRestrictions": null,
				    "resources": [
				        "namespaces"
				    ],
				    "verbs": [
				        "get",
				        "list"
				    ]
				]
			]
		]
	]

		def roleBinding = [
			apiVersion: "rbac.authorization.k8s.io/v1",
			kind: "RoleBinding",
			metadata: [
  				name: "broker-role-binding",
  				namespace: "${PROJECT_NAME}"
			],
			roleRef: [
  				apiGroup: "rbac.authorization.k8s.io",
  				kind: "Role",
  				name: "broker-role"
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
