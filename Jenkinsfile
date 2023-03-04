pipeline {
    agent any
    options {
        buildDiscarder(logRotator(daysToKeepStr: '5', numToKeepStr: '20'))
    }
    parameters {
        string defaultValue: '10',
               description: 'Please type the required number of users to be auto created in DB',
               name: 'USERS_TO_GENERATE_COUNT'
        choice choices: ['3' , '4', '3', '2', '1'],
               description: """Please choose number of kubernetes pods replicas count""",
               name: 'K8S_REPLICA_COUNT'
    }
    environment {
        FAILED_STAGE_NAME = ""
        COMPOSE_FILE = "docker-compose.yml"

        MYSQL_CONTAINER_NAME = "docker_app-mysql"
        PYTHON_CONTAINER_NAME = "docker_app-python"
        DOCKER_BACKEND_TESTING_CONTAINER_NAME = "docker_backend_testing-app"
        DB_CONTAINER_HOSTNAME = "mysql-0.mysql"
        PYTHON_CONTAINER_HOSTNAME = "python"
        DOCKER_BE_CONTAINER_HOSTNAME = "docker_backend_testing"
        PY_TAG = "py_app_ver_"
        PY_DOCKER_BE_TESTING_TAG = "py_docker_be_testing_app_ver_"
        MYSQL_DATABASE = "DevOpsExpertsDB"
        MYSQL_HOST_PORT = 5506
        MYSQL_GUEST_PORT = 3306
        RUN_SERVER = "backendOnly" // # .env file options that can be used for choosing which server to run: backendOnly / webOnly / bothServers
        CREATE_SCHEMA_FOR_K8S = "false" // # .env Boolean, if true - create default schema for k8s usage after pods are running.
        DOCKER_REPO = "amirhazan/devops_experts_proj"
        REMOTE_DB_HOST_ADDRESS = "devopsexpertsdb.cytk90qxc767.eu-south-1.rds.amazonaws.com"
        DOCKER_DB_HOST_ADDRESS = "mysql-0.mysql"
        REMOTE_DB_CONFIG_HOST = "127.0.0.1"
        DOCKER_DB_CONFIG_HOST = "0.0.0.0"
        K8S_MYSQL_FILE_NAME = "k8s-mysql-port.txt"
        K8S_FLASK_FILE_NAME = "k8s-flask-port.txt"
        HELM_CHART_NAMESPACE = "devops-experts-proj-ns"
        HELM_CHART_NAME = "devops-experts-proj-chart"
        MY_HELM_CHART_NAME = "devops-experts-proj"
        K8S_FLASK_SVC_NAME = "flask-app-svc"
        K8S_MYSQL_SVC_NAME = "mysql"
        K8S_TEST_TYPE = "k8s"
        DOCKER_TEST_TYPE = "docker"
        REMOTE_DB_TEST_TYPE = "remoteDB"
        REPLICA_COUNT = 3
    }
    stages {
        // stage # 1 GitHub checkout and set pollSCM - every 30min
        stage('checkout') {
            steps {
                script {
                    properties([pipelineTriggers([pollSCM('*/30 * * * *')])])
                }
                git credentialsId: 'githubCredentials', url: 'https://github.com/amir-hazan/DevOpsExpertsProj', branch: 'master'
            }
        }
        // stage # 2 install pip packages (overwrite if exist)
        stage('pip install') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    bat 'python -m pip install --ignore-installed --trusted-host pypi.python.org -r requirements.txt'
                }

            }
        }
        // stage # 3 update config.json with remote db credential (From Jenkins global credentials)
        stage('remote db - update credentials') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    withCredentials([usernamePassword(credentialsId: 'dbCredentials', passwordVariable: 'DB_PASS', usernameVariable: 'DB_USER_NAME')]) {
                        bat 'python src\\admin\\update_db_creds.py' + " " + updateConfigRemoteDB()
                    }
                }

            }
        }
        // stage # 4 DB tables will be created if not exist.
        stage('remote db - create db tables') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    bat 'python src\\db\\create_db_tables.py'
                }
            }
        }
        // stage # 5 Start flask rest server
        stage('remote db - run rest server') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    bat 'start /min python src\\rest_server\\rest_app.py'
                }
            }
        }
        // stage # 6 create 10 users in db, if pass 0 as value func will generate 10 users
        stage('remote db - create sample users') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    bat 'python src\\requests_api\\generate_new_users.py -t %REMOTE_DB_TEST_TYPE% -n ${params.USERS_TO_GENERATE_COUNT}'
                }
            }
        }
        // stage # 7 start backend testing
        stage('remote db - run backend testing') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    bat 'python src\\testings\\multi_platform_backend_testing.py -t %REMOTE_DB_TEST_TYPE%'
                }
            }
        }
        // stage # 8 stop flask rest server
        stage('remote db - stop rest server') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    bat 'python src\\requests_api\\get_stop_rest_server.py'
                }
            }
        }
        // stage # 9 DROP DB TABLES
        stage('remote db - drop db tables') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    bat 'python src\\db\\drop_db_tables.py'
                }
            }
        }
        // stage # 10 Clear db credentials from config.json (set as null)
        stage('remote db - clear db credentials') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    bat 'python src\\admin\\reset_db_creds.py'
                }
            }
        }
        // stage # 11 - parallel - create and set .env file and update config file
        stage('docker - create .env file && update config.json') {
            parallel {
                stage('docker - create & set .env file') {
                    steps {
                        script {
                            FAILED_STAGE_NAME = "${STAGE_NAME}"
                            withCredentials([usernamePassword(credentialsId: 'dockerMySqlRootCredentials', usernameVariable: 'MYSQL_ROOT_USER', passwordVariable: 'MYSQL_ROOT_PASSWORD'),
                                             usernamePassword(credentialsId: 'dbCredentials', usernameVariable: 'DB_USER_NAME', passwordVariable: 'DB_PASS')]) {
                                setEnvFile()
                            }
                        }
                    }
                }
                stage('docker - update config.json file') {
                    steps {
                        script {
                            FAILED_STAGE_NAME = "${STAGE_NAME}"
                            withCredentials([usernamePassword(credentialsId: 'dbCredentials', usernameVariable: 'DB_USER_NAME', passwordVariable: 'DB_PASS')]) {
                                bat 'python src\\admin\\update_db_creds.py' + " " + updateConfigForDockerAndK8SDB()
                            }
                        }
                    }
                }
            }
        }
        // stage # 12 - Login to docker-hub
        stage('docker hub login') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'HUB_PASS', usernameVariable: 'HUB_USERNAME')]) {
                        bat 'docker login --username "%HUB_USERNAME%" --password "%HUB_PASS%"'
                    }
                }
            }
        }
        // stage # 13 - docker-compose up + running flask server (using start.sh command code)
        stage('docker-compose up && build + backend-testing') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    bat 'docker-compose --env-file .env --file docker_app/%COMPOSE_FILE% up -d --build & docker ps -a'
                }
            }
        }
        // stage # 14 start backend testing
        stage('docker - create sample users') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    sleep(time: 10, unit: "SECONDS")
                    bat 'python src\\requests_api\\generate_new_users.py -t %DOCKER_TEST_TYPE% -n ${params.USERS_TO_GENERATE_COUNT}'
                }
            }
        }
        // stage # 15 start backend testing
        stage('docker - run backend testing (outside from the container)') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    sleep(time: 5, unit: "SECONDS")
                    bat 'python src\\testings\\multi_platform_backend_testing.py -t %DOCKER_TEST_TYPE%'
                }
            }
        }
        // stage # 16 - docker-compose push image to docker hub
        stage('docker-compose push to docker-hub') {
            steps {
                script {
                    bat 'docker-compose --env-file .env --file docker_app/%COMPOSE_FILE% push'
                }
            }
        }
        // stage # 17 - stop flask server
        stage('docker app - stop flask servers') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    sleep(time: 2, unit: "SECONDS")
                    bat "curl -i http://127.0.0.1:5000/admin/stop-rest-server"
                }
            }
        }
        // stage # 18 - clean docker environment
        stage('docker - clean environment') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    bat 'docker-compose --file docker_app/%COMPOSE_FILE% down --rmi all --volumes'
                }
            }
        }
        // stage # 19 - k8s install helm chart
        stage('k8s - config values.yaml') {
            parallel {
                stage ('k8s - set ReplicaCount') {
                    steps {
                        script {
                            FAILED_STAGE_NAME = "${STAGE_NAME}"
                            bat 'python src\\admin\\set_replica_count_in_helm_values.py -r ${params.K8S_REPLICA_COUNT}'
                        }
                    }
                }
                stage ('k8s - set build number') {
                    steps {
                        script {
                            FAILED_STAGE_NAME = "${STAGE_NAME}"
                            bat 'python src\\admin\\set_tag_ver_in_helm_values.py -b %BUILD_NUMBER%'
                        }
                    }
                }
            }
        }
        // stage # 20 - k8s start minikube
        stage('k8s - start minikube') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    bat 'minikube start'
                }
            }
        }
        // stage # 21 - k8s start minikube
        stage('k8s - create namespace') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    bat 'kubectl create namespace %HELM_CHART_NAMESPACE%'
                }
            }
        }
        // stage # 22 - k8s install helm chart
        stage('k8s - helm install') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    bat 'helm install %MY_HELM_CHART_NAME% %HELM_CHART_NAME% --namespace %HELM_CHART_NAMESPACE%'
                }
            }
        }
        // stage # 23 - k8s wait for pods
        stage('k8s - wait for pods') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    bat 'python src\\testings\\check_k8s_pods_state.py'
                    sleep(time: 15, unit: "SECONDS")
                }
            }
        }
        // stage # 24 - k8s install helm chart
        stage('k8s - get flask and sql ports') {
            parallel {
                stage ('k8s - save flask port') {
                    steps {
                        script {
                            FAILED_STAGE_NAME = "${STAGE_NAME}"
                            bat 'start /B minikube service %K8S_FLASK_SVC_NAME% --namespace %HELM_CHART_NAMESPACE% --url > %K8S_FLASK_FILE_NAME%'
                        }
                    }
                }
                stage ('k8s - save mysql port') {
                    steps {
                        script {
                            FAILED_STAGE_NAME = "${STAGE_NAME}"
                            bat 'start /B minikube service %K8S_MYSQL_SVC_NAME% --namespace %HELM_CHART_NAMESPACE% --url > %K8S_MYSQL_FILE_NAME%'
                        }
                    }
                }
            }
        }
        // stage # 25 - k8s backend testing
         stage('k8s - create sample users') {
             steps {
                 script {
                     FAILED_STAGE_NAME = "${STAGE_NAME}"
                     sleep(time: 10, unit: "SECONDS")
                     bat 'python src\\requests_api\\generate_new_users.py -t %K8S_TEST_TYPE% -n ${params.USERS_TO_GENERATE_COUNT}'
                 }
             }
         }
        // stage # 26 - k8s backend testing
        stage('k8s - run backend testing (outside from the container)') {
            steps {
                script {
                    FAILED_STAGE_NAME = "${STAGE_NAME}"
                    sleep(time: 5, unit: "SECONDS")
                    bat 'python src\\testings\\multi_platform_backend_testing.py -t %K8S_TEST_TYPE%'
                }
            }
        }
        // stage # 27 - k8s helm delete
        stage ('k8s - uninstall helm') {
            steps {
                script {
                    bat 'helm delete %MY_HELM_CHART_NAME% --namespace %HELM_CHART_NAMESPACE%'
                }
            }
        }
        // stage # 28 - k8s delete namespace
        stage ('k8s - delete namespace') {
            steps {
                script {
                    bat 'kubectl delete namespace %HELM_CHART_NAMESPACE%'
                }
            }
        }
        // stage # 29 - k8s stop minikube server
        stage ('k8s - stop minikube server') {
            steps {
                script {
                    bat 'minikube stop'
                }
            }
        }
        // stage # 30 - delete minikube and minikube docker image
        stage ('k8s - delete minikube') {
            steps {
                script {
                    bat 'minikube delete --purge'
                }
            }
        }
        // stage # 31 - delete minikube docker image
        stage ('k8s - delete minikube docker image') {
            steps {
                script {
                    def dockerImageId = getK8SDockerImageId()
                    echo "docker image id: " + dockerImageId
                    bat 'docker rmi -f ' + dockerImageId
                }
            }
        }
    }
    // start posting data based on result, if failure - send email.
    post {
        always {
            echo 'Pipeline done, checking if any failure for sending an email'
        }
        success {
            echo 'successfully done, no need to send email, see you next time.'
        }
        failure {
            emailext body: 'Job name: ' + "${env.JOB_NAME}" + ' pipeline<br><br>' + 'Build URL: ' + "${env.BUILD_URL}" + '<br><br> fail on stage: ' + "${FAILED_STAGE_NAME}",
                    subject: 'CI FAIL: ' + "${env.JOB_NAME}" + ' pipeline',
                    to: '$DEFAULT_RECIPIENTS'
        }
        unstable {
            echo 'run was marked as unstable'
        }
        changed {
            echo 'Pipeline was changed from last run, please follow logs.'
        }
    }
}

def setEnvFile() {
    bat 'echo BUILD_NUMBER=%BUILD_NUMBER% > .env'
    bat 'echo MYSQL_CONTAINER_NAME=%MYSQL_CONTAINER_NAME% >> .env'
    bat 'echo PYTHON_CONTAINER_NAME=%PYTHON_CONTAINER_NAME% >> .env'
    bat 'echo DOCKER_BACKEND_TESTING_CONTAINER_NAME=%DOCKER_BACKEND_TESTING_CONTAINER_NAME% >> .env'
    bat 'echo DB_CONTAINER_HOSTNAME=%DB_CONTAINER_HOSTNAME% >> .env'
    bat 'echo PYTHON_CONTAINER_HOSTNAME=%PYTHON_CONTAINER_HOSTNAME% >> .env'
    bat 'echo DOCKER_BE_CONTAINER_HOSTNAME=%DOCKER_BE_CONTAINER_HOSTNAME% >> .env'
    bat 'echo PY_TAG=%PY_TAG% >> .env'
    bat 'echo PY_DOCKER_BE_TESTING_TAG=%PY_DOCKER_BE_TESTING_TAG% >> .env'
    bat 'echo MYSQL_ROOT_USER=%MYSQL_ROOT_USER% >> .env'
    bat 'echo MYSQL_ROOT_PASSWORD=%MYSQL_ROOT_PASSWORD% >> .env'
    bat 'echo MYSQL_DATABASE=%MYSQL_DATABASE% >> .env'
    bat 'echo MYSQL_USER=%DB_USER_NAME% >> .env'
    bat 'echo MYSQL_PASSWORD=%DB_PASS% >> .env'
    bat 'echo MYSQL_HOST_PORT=%MYSQL_HOST_PORT% >> .env'
    bat 'echo MYSQL_GUEST_PORT=%MYSQL_GUEST_PORT% >> .env'
    bat 'echo RUN_SERVER=%RUN_SERVER% >> .env'
    bat 'echo CREATE_SCHEMA_FOR_K8S=%CREATE_SCHEMA_FOR_K8S% >> .env'
    bat 'echo HELM_CHART_NAMESPACE=%HELM_CHART_NAMESPACE% >> .env'
}

def updateConfigForDockerAndK8SDB() {
    return "-a %DOCKER_DB_HOST_ADDRESS% -u \"%DB_USER_NAME%\" -p \"%DB_PASS%\" -c %DOCKER_DB_CONFIG_HOST%"
}

def updateConfigRemoteDB() {
    return "-a %REMOTE_DB_HOST_ADDRESS% -u \"%DB_USER_NAME%\" -p \"%DB_PASS%\" -c %REMOTE_DB_CONFIG_HOST%"
}

def getK8SDockerImageId() {
    def dockerImageId = bat(script: 'docker images --quiet gcr.io/k8s-minikube/kicbase', returnStdout: true).trim().readLines().drop(1).join(" ").replaceAll("\'","");
    return dockerImageId
}
