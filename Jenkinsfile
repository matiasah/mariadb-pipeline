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
        string(description: "Root User name", name: "ROOT_USERNAME", defaultValue: env.ROOT_USERNAME ? env.ROOT_USERNAME : '')
        string(description: "Root User password", name: "ROOT_USERPASSWORD", defaultValue: env.ROOT_USERPASSWORD ? env.ROOT_USERPASSWORD : '')
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
                        sh "helm repo add bitnami https://charts.bitnami.com/bitnami"
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
                        sh "kubectl create namespace mariadb --dry-run=client -o yaml | kubectl apply -f -"

                    }

                }

            }

        }

        stage ("Template: Plan") {

            steps {

                container ("helm") {

                    script {

                        // Cluster agent options
                        MARIADB_OPTIONS = " "

                        // If DEBUG is enabled
                        if (env.DEBUG.equals("true")) {

                            // Enable debug
                            MARIADB_OPTIONS += "--debug "

                        }

                        // If Root user is provided
                        if (env.ROOT_USERNAME && env.ROOT_USERPASSWORD) {

                            // Provide root user name and password
                            MARIADB_OPTIONS += "--set rootUser.user=${env.ROOT_USERNAME},rootUser.password=${env.ROOT_USERPASSWORD} "

                        }

                        // Template
                        sh "helm template mariadb bitnami/mariadb-galera -f mariadb-values.yaml ${MARIADB_OPTIONS.trim()} --namespace mariadb > mariadb-template.yaml"

                        // Print Yaml
                        sh "cat mariadb-template.yaml"
        
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
                            sh "kubectl apply -f mariadb-template.yaml"

                        // Destroy
                        } else if (env.ACTION.equals("Destroy")) {

                            try {
                                
                                // Destroy
                                sh "kubectl delete -f mariadb-template.yaml"

                            } catch (Exception e) {

                                // Do nothing

                            }

                        }

                        sh "rm mariadb-template.yaml"

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

                        sh "kubectl rollout restart deployment -n mariadb"
                        sh "kubectl rollout restart daemonset -n mariadb"
                        sh "kubectl rollout restart statefulset -n mariadb"

                    }

                }

            }

        }

    }
    
}
