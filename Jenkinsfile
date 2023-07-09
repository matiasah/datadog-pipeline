/*
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-slave
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-slave-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins-slave
  namespace: jenkins
*/

pipeline {

    parameters {
        choice(description: "Action", name: "Action", choices: ["Plan", "Apply"])
        string(description: "Cluster Name", name: "CLUSTER_NAME", defaultValue: env.CLUSTER_NAME ? env.CLUSTER_NAME : '')
        choice(description: "Log Level", name: "LOG_LEVEL", choices: ["ERROR", "INFO", "DEBUG", "CRITICAL", "OFF"])
        credentials(description: "DataDog API Key", name: "API_KEY", defaultValue: env.API_KEY ? env.API_KEY : '', credentialType: "Token", required: true)
        booleanParam(description: "Deploy Pod Security Policy", name: "DEPLOY_CLUSTER_AGENT_PSP", defaultValue: true)
        booleanParam(description: "Debug", name: "DEBUG", defaultValue: env.DEBUG ? env.DEBUG : "false")
    }

    agent {
        kubernetes {
            yaml """
                apiVersion: "v1"
                kind: "Pod"
                spec:
                  securityContext:
                    runAsUser: 1001
                    runAsGroup: 1001
                    fsGroup: 1001
                  containers:
                  - command:
                    - "cat"
                    image: "alpine/helm:latest"
                    imagePullPolicy: "IfNotPresent"
                    name: "helm"
                    resources: {}
                    tty: true
                  - command:
                    - "cat"
                    image: "bitnami/kubectl:latest"
                    imagePullPolicy: "IfNotPresent"
                    name: "kubectl"
                    resources: {}
                    tty: true
                  serviceAccountName: jenkins-slave
            """
        }
    }

    stages {

        stage ("Helm Repo") {

            steps {

                container ("helm") {

                    script {
    
                        // Install repo
                        sh "helm repo add datadog https://helm.datadoghq.com"
                        sh "helm repo update"
    
                    }

                }

            }

        }

        stage ("API Key: Apply") {

            when {
                
                expression {
                    return env.ACTION.equals("Apply") || env.ACTION.equals("Destroy")
                }
                
            }

            steps {

                container ("kubectl") {

                    script {

                        // Apply
                        if (env.ACTION.equals("Apply")) {

                            // Apply API Key
                            sh "kubectl create secret generic datadog-secret --namespace datadog --from-literal api-key=${API_KEY} --dry-run=client -o yaml | kubectl apply -f -"

                        // Destroy
                        } else if (env.ACTION.equals("Destroy")) {

                            try {

                                // Destroy API Key
                                sh "kubectl delete secret datadog-secret --namespace datadog"

                            } catch (Exception e) {

                                // Do nothing
                                
                            }

                        }

                    }

                }

            }

        }

        stage ("Template: Plan") {

            steps {

                container ("helm") {

                    script {
        
                    }

                }

            }

        }

        stage ("Template: Apply") {

            when {
                
                expression {
                    return env.ACTION.equals("Apply")
                }
                
            }

            steps {

                container ("kubectl") {

                    script {

                    }

                }

            }

        }

    }
    
}
