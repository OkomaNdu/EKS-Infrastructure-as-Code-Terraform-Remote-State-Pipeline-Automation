#!/usr/bin/env groovy

pipeline {
    agent any
    environment {
        AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws_secret_access_key')
    }
    stages {
        stage('provision cluster') {
            environment {
                TF_VAR_env_prefix = "dev"
                TF_VAR_k8s_version = "1.35"
                TF_VAR_cluster_name = "my-cluster"
                TF_VAR_region = "ca-central-1"
            }
            steps {
                script {
                    echo "creating EKS cluster"
                    sh "terraform init"
                    sh "terraform apply --auto-approve"
                    
                    env.K8S_CLUSTER_URL = sh(
                        script: "terraform output cluster_url",
                        returnStdout: true
                    ).trim()
                    
                    // set kubeconfig access
                    sh "aws eks update-kubeconfig --name ${TF_VAR_cluster_name} --region ${TF_VAR_region}"
                }
            }
        }
    }
}