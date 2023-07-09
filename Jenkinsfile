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
        choice(description: "Action", name: "Action", choices: ["Plan", "Apply", "Destroy"])
        string(description: "Cluster Name", name: "CLUSTER_NAME", defaultValue: env.CLUSTER_NAME ? env.CLUSTER_NAME : '')
        choice(description: "Log Level", name: "LOG_LEVEL", choices: ["ERROR", "INFO", "DEBUG", "CRITICAL", "OFF"])
        choice(description: "Target System", name: "TARGET_SYSTEM", choices: ["linux", "windows"])
        credentials(description: "DataDog API Key", name: "API_KEY_CREDENTIAL_ID", defaultValue: env.API_KEY_CREDENTIAL_ID ? env.API_KEY_CREDENTIAL_ID : '', credentialType: "Secret text", required: true)
        booleanParam(description: "Deploy Pod Security Policy", name: "DEPLOY_CLUSTER_AGENT_PSP", defaultValue: env.DEPLOY_CLUSTER_AGENT_PSP ? env.DEPLOY_CLUSTER_AGENT_PSP : false)
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
                    volumeMounts:
                    - mountPath: "/.config"
                      name: "config-volume"
                      readOnly: false
                    - mountPath: "/.cache/helm/"
                      name: "cache-volume"
                      readOnly: false
                  - command:
                    - "cat"
                    image: "bitnami/kubectl:latest"
                    imagePullPolicy: "IfNotPresent"
                    name: "kubectl"
                    resources: {}
                    tty: true
                  serviceAccountName: jenkins-slave
                  volumes:
                  - emptyDir:
                      medium: ""
                    name: "config-volume"
                  - emptyDir:
                      medium: ""
                    name: "cache-volume"
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

        stage ("Namespace: Apply") {

            when {

                expression {
                    return env.ACTION.equals("Apply")
                }

            }

            steps {

                container ("kubectl") {

                    script {

                        // Apply
                        sh "kubectl create namespace datadog --dry-run=client -o yaml | kubectl apply -f -"

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

                            // Get Credential
                            withCredentials([string(credentialsId: env.API_KEY_CREDENTIAL_ID, variable: 'API_KEY')]) {

                                // Apply API Key
                                sh "kubectl create secret generic datadog-secret --namespace datadog --from-literal api-key=${API_KEY} --dry-run=client -o yaml | kubectl apply -f -"

                            }

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

                        // Cluster agent options
                        CLUSTER_AGENT_OPTIONS = "--set datadog.logLevel=${LOG_LEVEL} "
                        CLUSTER_AGENT_OPTIONS = "--set targetSystem=${TARGET_SYSTEM} "
                        CLUSTER_AGENT_OPTIONS += "--set datadog.clusterName=${CLUSTER_NAME} "

                        // If Pod Security Policy is enabled
                        if (env.DEPLOY_CLUSTER_AGENT_PSP.equals("true")) {

                            // Enable Pod Security Policy
                            CLUSTER_AGENT_OPTIONS += "--set agents.podSecurity.podSecurityPolicy.create=true --set clusterAgent.podSecurity.podSecurityPolicy.create=true --api-versions \"policy/v1beta1/PodSecurityPolicy\" "

                        }

                        // Template
                        sh "helm template datadog -f cluster-agent-values.yaml datadog/datadog ${CLUSTER_AGENT_OPTIONS} --namespace datadog > datadog-cluster-agent.yaml"

                        // Print Yaml
                        sh "cat datadog-cluster-agent.yaml"
        
                    }

                }

            }

        }

        stage ("Template: Apply") {

            when {
                
                expression {
                    return !env.ACTION.equals("Plan")
                }
                
            }

            steps {

                container ("kubectl") {

                    script {

                        // Apply
                        if (env.ACTION.equals("Apply")) {

                            // Apply
                            sh "kubectl apply -f datadog-cluster-agent.yaml"

                        // Destroy
                        } else if (env.ACTION.equals("Destroy")) {

                            try {
                                
                                // Destroy
                                sh "kubectl delete -f datadog-cluster-agent.yaml"

                            } catch (Exception e) {

                                // Do nothing

                            }

                        }

                        sh "rm datadog-cluster-agent.yaml"

                    }

                }

            }

        }

        stage ("Restart") {

            when {
                
                expression {
                    return !env.ACTION.equals("Plan")
                }
                
            }

            steps {

                container ("kubectl") {

                    script {

                        sh "kubectl rollout restart deployment -n datadog"
                        sh "kubectl rollout restart daemonset -n datadog"

                    }

                }

            }

        }

    }
    
}
