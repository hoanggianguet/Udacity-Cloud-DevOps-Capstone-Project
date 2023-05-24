pipeline {
	
	agent any

	environment {
    	DOCKERHUB_CREDENTIALS = credentials('dockerhub')
  	}

	stages {
		
		stage("Lint HTML static files") {
			steps {
				sh 'tidy -q -e ./blue-app/*.html'
				sh 'tidy -q -e ./green-app/*.html'
			}
		}
		stage("Adding Script Permissions") {
			steps {
				sh 'echo " --- change file permission to executable --- "'
				sh '''
					cd ./blue-app
					chmod +x ./build_docker.sh
					chmod +x ./upload_docker.sh
					chmod +x ./remove_docker.sh
					chmod +x ./blue-app.yaml
					chmod +x ./blue-service.yaml
				'''
				sh '''
					cd ./green-app
					chmod +x ./build_docker.sh
					chmod +x ./upload_docker.sh
					chmod +x ./remove_docker.sh
					chmod +x ./green-app.yaml
					chmod +x ./green-service.yaml
				'''
			}
		}
		stage("Build Docker Images") {
			parallel {
				stage("Build Blue Image") {
					steps {
						sh 'echo " ---- Building Blue Image --- "'
						sh '''
							echo $BUILD_ID
							cd ./blue-app
							./build_docker.sh
						'''						
					}
				}
				stage("Build Green Image") {
					steps {	
						sh 'echo " ---- Building Green Image --- "'
						sh '''
							echo $BUILD_ID
							cd ./green-app
							./build_docker.sh
						'''
					}
				}
			}
		}
		
		stage("Push Docker Images to DockerHub") {
			parallel {
				stage("Push Blue Image") {
					steps {
						sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
						sh '''
							cd ./blue-app
							./upload_docker.sh
						'''
					}
				}
				stage("Push Green Image") {
					steps {
						sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
						sh '''
							cd ./green-app
							./upload_docker.sh
						'''
					}
				}
			}
		}
		
		stage ("Remove Docker Images") {
			parallel {
				stage("Remove Blue Image") {
					steps {
						sh 'echo " ---- Removing Blue Image --- "'
						sh '''
							cd ./blue-app
							./remove_docker.sh
						'''
					}
				}
				stage("Remove Green Image") {
					steps {
						sh 'echo " ---- Removing Green Image --- "'
						sh '''
							cd ./green-app
							./remove_docker.sh
						'''
					}
				}
			}
		}
		stage("Create Kubernetes Cluster") {
			steps {
				withAWS(region:'us-east-1',credentials:'aws-jenkins') {
					sh 'eksctl create cluster --name capstone-project --version 1.26 --region us-east-1 --nodegroup-name project-udacity --node-type t3.micro --nodes 4 --nodes-min 2 --nodes-max 4 --managed'
				}
			}
		}
		stage("Update K8s Cluster Context") {
			steps {
				withAWS(region:'us-east-1',credentials:'aws-jenkins') {
					sh 'aws eks --region us-east-1 update-kubeconfig --name capstone-project '
				}
			}
		}
		stage("Deploy Application Containers") {
			parallel {
				stage("Deploy Blue Application Container") {
					steps {
						withAWS(region:'us-east-1',credentials:'aws-jenkins') {
							sh '''
								cd ./blue-app
								kubectl apply -f blue-app.yaml
							'''
						}
					}
				}
				stage("Deploy Green Application Container") {
					steps {
						withAWS(region:'us-east-1',credentials:'aws-jenkins') {
							sh '''
								cd ./green-app
								kubectl apply -f green-app.yaml
							'''
						}
					}
				}
			}
		}
		stage("Run Blue Application") {
			steps {
				withAWS(region:'us-east-1',credentials:'aws-jenkins') {
					sh '''
						cd ./blue-app
						kubectl apply -f blue-service.yaml
					'''
				}
			}
		}
		stage("Blue Application") {
			steps {
				withAWS(region:'us-east-1',credentials:'aws-jenkins') {
					sleep time: 1, unit: 'MINUTES'
					sh 'kubectl get service -o wide'
					sleep time: 1, unit: 'MINUTES'
				}
			}
		}
		stage("Switch to Green Application") {
			steps {
				withAWS(region:'us-east-1',credentials:'aws-jenkins') {
					sh 'kubectl apply -f green-app/green-service.yaml'
				}
			}
		}
		stage("Green Application") {
			steps {
				withAWS(region:'us-east-1',credentials:'aws-jenkins') {
					sleep time: 1, unit: 'MINUTES'
					sh 'kubectl get service -o wide'
				}
			}
		}
		
	}
}