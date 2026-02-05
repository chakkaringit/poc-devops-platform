pipeline {
    agent any

    parameters {
        string(name: 'INPUT_CLUSTER_NAME', defaultValue: 'My-Jenkins-Cluster', description: 'EKS Cluster Name (Must be unique)')
        choice(name: 'INPUT_INSTANCE_TYPE', choices: ['t3.medium', 't3.large','t4g.xlarge', 'm5.large', 'm5.xlarge'], description: 'EC2 Worker Node Size')
        choice(name: 'ACTION', choices: ['Deploy', 'Destroy'], description: 'Action to perform')
        string(name: 'INPUT_WILDCARD_DOMAIN', defaultValue: '*.example.com', description: 'Wildcard Domain Name (e.g. *.yourshop.com)')
        string(name: 'INPUT_SUB_DOMAIN', defaultValue: 'knowesis.prod.example.com', description: 'Sub Domain Name (e.g. knowesis.prod.example.com)')
        string(name: 'INPUT_CERT_ARN', defaultValue: '', description:'ARN of ACM Certificate (Must be in Region us-east-1)')
    }

    environment {
        AWS_REGION = 'ap-southeast-1'
        
        // ⚠️ เช็คว่าไฟล์นี้อยู่ใน Folder provisioning ใน Git จริงหรือไม่
        TEMPLATE_FILE = 'provisioning/eks-full-stack.yaml' 
        SECURITY_TEMPLATE_FILE = 'provisioning/edge-security/cloudfront-waf-stack.yaml' 
        STACK_NAME = "EKS-${params.INPUT_CLUSTER_NAME}-Stack" 
        
        // ชื่อ ID ของ Credential ใน Jenkins
        AWS_CRED_ID = 'maas-aws-key-main'
    }

    stages {
        // Stage 1: Checkout
        stage('Checkout Code') {
            steps {
                script {
                   // ล้าง Workspace เก่าก่อนเพื่อความสะอาด
                   cleanWs()
                }
                
                // ดึง Code จาก GitHub
                git branch: 'main', 
                    url: 'https://github.com/chakkaringit/poc-devops-platform.git'
            }
        }

        // Stage 2: Verify YAML
        stage('Validate Template') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CRED_ID, accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    script {
                        echo "Validating CloudFormation Template..."
                        // เช็คว่าไฟล์มีอยู่จริงไหม ก่อนจะ validate
                        sh "ls -la provisioning/" 
                        sh "aws cloudformation validate-template --template-body file://${TEMPLATE_FILE} --region ${AWS_REGION}"
                    }
                }
            }
        }

        // Stage 3: Deploy EKS Cluster
        stage('Deploy Infrastructure') {
            when {
                expression { params.ACTION == 'Deploy' }
            }
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CRED_ID, accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    script {
                        echo "============================================="
                        echo "Deploying Stack: ${STACK_NAME}"
                        echo "Cluster Name: ${params.INPUT_CLUSTER_NAME}"
                        echo "Instance Type: ${params.INPUT_INSTANCE_TYPE}"
                        echo "============================================="
                        
                        sh """
                            aws cloudformation deploy \
                            --template-file ${TEMPLATE_FILE} \
                            --stack-name ${STACK_NAME} \
                            --region ${AWS_REGION} \
                            --capabilities CAPABILITY_NAMED_IAM \
                            --parameter-overrides \
                                ClusterName=${params.INPUT_CLUSTER_NAME} \
                                NodeInstanceType=${params.INPUT_INSTANCE_TYPE}
                        """
                        
                        echo "Deployment Completed Successfully!"
                    }
                }
            }
        }

        // Stage 4: Install ArgoCD
        stage('Install ArgoCD') {
            when {
                expression { params.ACTION == 'Deploy' }
            }
            steps {
                script {
                    try{
                        echo "Installing ArgoCD..."
                        def errorDesc = ""
                        def customKubeConfig = "${WORKSPACE}/.kubeconfig-temp"
                    
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CRED_ID, accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        
                            sh "aws eks update-kubeconfig --name ${params.INPUT_CLUSTER_NAME} --region ${AWS_REGION} --kubeconfig ${customKubeConfig}"
                        
                            withEnv(["KUBECONFIG=${customKubeConfig}"]) {
                                // 1. Install ArgoCD (เหมือนเดิม)
                                sh "kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -"
                                sh "kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts"
                                //sh "kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml"
                                
                                // เราใช้ cat <<EOF เพื่อเขียนไฟล์ YAML สดๆ ลงไปเลย
                                errorDesc = " : Create ArgoCD ingress exception !"
                                sh """
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: nginx
    # สำคัญ: ArgoCD รันเป็น HTTPS ภายใน เราต้องบอก NGINX ให้คุยแบบ HTTPS
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # รองรับการส่งผ่านไฟล์ใหญ่ๆ (เผื่อ deploy manifest ใหญ่)
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
spec:
  rules:
  # ตั้งชื่อโดเมนย่อยสำหรับ ArgoCD (เช่น argocd.poc.quiinsfelicity.shop)
  # ใช้ replace เพื่อตัด *. ออก (ถ้ามี)
  - host: argocd.${params.INPUT_SUB_DOMAIN.replace('*.', '')}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
        EOF
"""

                                echo "Waiting for ArgoCD..."
                                sleep 10
                                sh "kubectl wait --namespace argocd --for=condition=ready pod --selector=app.kubernetes.io/name=argocd-server --timeout=180s"
                            }
                        }
                    
                    } catch (Exception e){
                        currentBuild.result = 'FAILURE'
                        currentBuild.description = "Error: ${e.message}"
                        throw e
                    }
                }  
            }
        }

        // Stage 5: Install Ingress Controller
        stage('Install Ingress Controller') {
            when {
                expression { params.ACTION == 'Deploy' }
            }
            steps {
                script {
                    echo "Installing NGINX Ingress Controller..."
                    def customKubeConfig = "${WORKSPACE}/.kubeconfig-temp"
                    
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CRED_ID, accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        
                        sh """
                            aws eks update-kubeconfig \
                            --name ${params.INPUT_CLUSTER_NAME} \
                            --region ${AWS_REGION} \
                            --kubeconfig ${customKubeConfig}
                        """
                        
                        withEnv(["KUBECONFIG=${customKubeConfig}"]) {
                            sh "kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/aws/deploy.yaml"
                            
                            echo "Waiting for Ingress Controller to be ready..."
                            sleep 10
                            sh "kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=120s"
                            
                            echo "Ingress Controller Installed Successfully!"
                        }
                    }
                }
            }
        }
        
        // Stage 6: Deploy CloudFront + WAF
        stage('Deploy CloudFront + WAF') {
            when {
                expression { params.ACTION == 'Deploy' }
            }
            steps {
                script {
                    echo "Deploying CloudFront & WAF..."
                    def customKubeConfig = "${WORKSPACE}/.kubeconfig-temp"
                    
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CRED_ID, accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        
                        // 1. ดึงชื่อ NLB URL
                        sh "aws eks update-kubeconfig --name ${params.INPUT_CLUSTER_NAME} --region ${AWS_REGION} --kubeconfig ${customKubeConfig}"
                        
                        def nlbUrl = ""
                        withEnv(["KUBECONFIG=${customKubeConfig}"]) {
                            nlbUrl = sh(script: "kubectl get service ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()
                        }
                        
                        if (nlbUrl == "") {
                            error "Could not find NLB Hostname!"
                        }
                        echo "Found NLB URL: ${nlbUrl}"

                        // 2. Deploy CloudFront Stack ไปที่ us-east-1
                        // ⚠️ เช็ค Path ไฟล์นี้ใน Git ให้ชัวร์นะครับ
                        echo "Deploying Stack to us-east-1..."
                        sh """
                            aws cloudformation deploy \
                            --template-file ${SECURITY_TEMPLATE_FILE} \
                            --stack-name Edge-Security-${params.INPUT_CLUSTER_NAME} \
                            --region us-east-1 \
                            --capabilities CAPABILITY_NAMED_IAM \
                            --parameter-overrides \
                                NlbDomainName=${nlbUrl} \
                                WildcardDomain=${params.INPUT_WILDCARD_DOMAIN} \
                                AcmCertificateArn=${params.INPUT_CERT_ARN}
                        """
                        
                        echo "CloudFront Deployment Started! (Check CloudFormation Console in us-east-1)"
                    }
                }
            }
        }
        
        // Stage 7: Install EFS Driver
        stage('Install EFS Driver & StorageClass') {
            when { expression { params.ACTION == 'Deploy' } }
            steps {
                script {
                   def customKubeConfig = "${WORKSPACE}/.kubeconfig-temp"
                   
                   withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CRED_ID, accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                       
                       // 1. ดึงค่า EFS ID จาก CloudFormation Output (ท่าไม้ตาย!)
                       // คำสั่งนี้จะไปถาม AWS ว่า Stack นี้สร้าง EFS ID อะไรมา
                       def efsId = sh(script: "aws cloudformation describe-stacks --stack-name ${STACK_NAME} --region ${AWS_REGION} --query 'Stacks[0].Outputs[?OutputKey==`EFSFileSystemId`].OutputValue' --output text", returnStdout: true).trim()
                       
                       echo "Found EFS FileSystem ID: ${efsId}"
                       
                       if (efsId == "" || efsId == "None") {
                           error "Could not find EFS ID. Did you update CloudFormation to include EFS?"
                       }

                       // 2. Login EKS
                       sh "aws eks update-kubeconfig --name ${params.INPUT_CLUSTER_NAME} --region ${AWS_REGION} --kubeconfig ${customKubeConfig}"
                       
                       withEnv(["KUBECONFIG=${customKubeConfig}"]) {
                           echo "Installing EFS CSI Driver..."
                           sh "kubectl apply -k 'github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master'"
                           
                           // 3. สร้าง StorageClass (Inject ID เข้าไปสดๆ)
                           echo "Creating StorageClass with ID: ${efsId}..."
                           sh """
cat <<EOF | kubectl apply -f -
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: ${efsId}  # <--- Jenkins เติมค่าให้ตรงนี้
  directoryPerms: "777"
  uid: "1000"
  gid: "1000"
EOF
"""
                       }
                   }
                }
            }
        }
        
        // Stage 7: Destroy
        stage('Destroy Infrastructure') {
            when {
                expression { params.ACTION == 'Destroy' }
            }
            steps {
                script {
                    input message: "⚠️ DANGER ZONE! ⚠️\nยืนยันที่จะทำลายระบบทั้งหมดของ: ${params.INPUT_CLUSTER_NAME} ?", ok: "ยืนยันการลบถาวร"

                    def customKubeConfig = "${WORKSPACE}/.kubeconfig-temp"
                    
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: AWS_CRED_ID, accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        
                        // Step 1: ลบ CloudFront (us-east-1)
                        echo "1. Destroying Edge Security (CloudFront/WAF)..."
                        sh """
                            aws cloudformation delete-stack \
                            --stack-name Edge-Security-${params.INPUT_CLUSTER_NAME} \
                            --region us-east-1
                        """
                        echo "Delete command sent for CloudFront Stack."

                        // Step 2: ลบ Kubernetes Load Balancers
                        echo "2. Cleaning up Kubernetes Load Balancers..."
                        try {
                            sh "aws eks update-kubeconfig --name ${params.INPUT_CLUSTER_NAME} --region ${AWS_REGION} --kubeconfig ${customKubeConfig}"
                            withEnv(["KUBECONFIG=${customKubeConfig}"]) {
                                sh "kubectl delete service ingress-nginx-controller -n ingress-nginx --ignore-not-found=true"
                                sh "kubectl delete service argocd-server -n argocd --ignore-not-found=true"
                                echo "Waiting 2 minutes for AWS to clean up NLB..."
                                sleep 120 
                            }
                        } catch (Exception e) {
                            echo "⚠️ Warning: Could not connect to cluster. Proceeding..."
                        }

                        // Step 3: ลบ EKS Stack
                        echo "3. Destroying Main EKS Stack..."
                        sh "aws cloudformation delete-stack --stack-name ${STACK_NAME} --region ${AWS_REGION}"
                        sh "aws cloudformation wait stack-delete-complete --stack-name ${STACK_NAME} --region ${AWS_REGION}"
                        
                        echo "✅ All Systems Destroyed Successfully."
                    }
                }
            }
        }
    } // End of stages

    post {
        failure {
            echo "Pipeline Failed. Please check the logs."
        }
    }
} // End of pipeline