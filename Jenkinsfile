pipeline {
    agent any           // Tools olarak terraform tanimliyoruz. 
    tools {                             
        terraform 'terraform'
}
    environment {       // env lar Stages lerin altinda da tanimlanir ama sade orada kullanirsin bu sekilde heryerde kullanabilirsin.
        PATH=sh(script:"echo $PATH:/usr/local/bin", returnStdout:true).trim()       //AWS cli komutlarinin caslimasi icin tanimliyoruz. 
        AWS_REGION = "us-east-1"
        AWS_ACCOUNT_ID=sh(script:'export PATH="$PATH:/usr/local/bin" && aws sts get-caller-identity --query Account --output text', returnStdout:true).trim()
        ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        APP_REPO_NAME = "jenkins-repo/phonebook-app"            // repo ismini dosyada degistirdiysen burada da degistirmelisin
        APP_NAME = "phonebook"
        CFN_KEYPAIR="neu"
        HOME_FOLDER = "/home/ec2-user"
        GIT_FOLDER = sh(script:'echo ${GIT_URL} | sed "s/.*\\///;s/.git$//"', returnStdout:true).trim()     // Buradan git repo ismini aliyor sadece.
    }
    stages {
        stage('Create ECR Repo') {                  // MUTABLE ecr degistirilebilir. Yani üzerine yazilabilir yoksa devamli ayni isimde gönderemiyoruz.          
            steps {
                echo 'Creating ECR Repo for App'
                sh """
                aws ecr create-repository \
                  --repository-name ${APP_REPO_NAME} \
                  --image-scanning-configuration scanOnPush=false \
                  --image-tag-mutability MUTABLE \                  
                  --region ${AWS_REGION}
                """
            }
        }

        stage('Build App Docker Image') {           //docker file oldugu yerde image ondan olusturuyor. --force-rm ile olusturdugu gecici (intermedit) conterlari siliyor.
            steps {
                echo 'Building App Image'
                sh 'docker build --force-rm -t "$ECR_REGISTRY/$APP_REPO_NAME:latest" .'
                sh 'docker image ls'
            }
        }

        stage('Push Image to ECR Repo') {           //aws ecr girebilmek icin password alip. Baglanacagi ECR name login oluyor.
            steps {
                echo 'Pushing App Image to ECR Repo'
                sh 'aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:latest"'
            }
        }

        stage('Create Infrastructure for the App') {        //terraform file daki docker-swarm Ec2 larinin dogru calismasi yapilandiriliyor.
            steps {
                echo 'Creating Infrastructure for the App on AWS Cloud'
                sh 'terraform init'
                sh 'terraform apply --auto-approve'
                script {
                    echo 'Waiting for Leader Manager'
                    id = sh(script: 'aws ec2 describe-instances --filters Name=tag-value,Values=docker-grand-master Name=instance-state-name,Values=running --query Reservations[*].Instances[*].[InstanceId] --output text',  returnStdout:true).trim()
                    sh 'aws ec2 wait instance-status-ok --instance-ids $id'

                    echo 'Leader manager is running'
                    mid = sh(script: 'aws ec2 describe-instances --filters Name=tag-value,Values=docker-manager-2 Name=instance-state-name,Values=running --query Reservations[*].Instances[*].[InstanceId] --output text',  returnStdout:true).trim()
                    sh 'aws ec2 wait instance-status-ok --instance-ids $mid'
                    
                    wid = sh(script: 'aws ec2 describe-instances --filters Name=tag-value,Values=docker-worker-1 Name=instance-state-name,Values=running --query Reservations[*].Instances[*].[InstanceId] --output text',  returnStdout:true).trim()
                    sh 'aws ec2 wait instance-status-ok --instance-ids $wid'
                    env.MASTER_INSTANCE_PUBLIC_IP = sh(script:'aws ec2 describe-instances --region ${AWS_REGION} --filters Name=tag-value,Values=docker-grand-master Name=instance-state-name,Values=running --query Reservations[*].Instances[*].[PublicIpAddress] --output text | sed "s/\\s*None\\s*//g"', returnStdout:true).trim()  
                }
            }
        }

        stage('Test the Infrastructure') {          //Burada yapiyi test ediyoruz. Burada döngü kullaniyoruz. Farkli seylerde var bakilabilir.
                                                    //Viz app den yanit almayi bekliyor. Yanitgelirse devam ediyor. Gelmezse bekliyor.
             steps {
                 echo "Testing if the Docker Swarm is ready or not, by checking Viz App on Grand Master with Public Ip Address: ${MASTER_INSTANCE_PUBLIC_IP}:8080"
                 script {
                 while(true) {

                 try {
                    sh "curl -s --connect-timeout 60 ${MASTER_INSTANCE_PUBLIC_IP}:8080"
                    echo "Successfully connected to Viz App."
                    break
                    }
                 catch(Exception) {
                    echo 'Could not connect Viz App'
                    sleep(5)
                     }
                   }
                 }
             }
         }

        stage('Deploy App on Docker Swarm'){            //Master baglanip repoyu indiriyor. Sonra docker compose calistiriyor.
            environment {
                MASTER_INSTANCE_ID=sh(script:'aws ec2 describe-instances --region ${AWS_REGION} --filters Name=tag-value,Values=docker-grand-master Name=instance-state-name,Values=running --query Reservations[*].Instances[*].[InstanceId] --output text', returnStdout:true).trim()
            }
            steps {

                echo "Cloning and Deploying App on Swarm using Grand Master with Instance Id: $MASTER_INSTANCE_ID"
                sh 'mssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no --region ${AWS_REGION} ${MASTER_INSTANCE_ID} git clone ${GIT_URL}'
                sleep(20)
                sh 'mssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no --region ${AWS_REGION} ${MASTER_INSTANCE_ID} docker stack deploy --with-registry-auth -c ${HOME_FOLDER}/${GIT_FOLDER}/docker-compose.yml ${APP_NAME}'
            }
        }

        stage('Destroy the infrastructure'){    //5 gün birsey yapmazsaniz silecek. Dokümantasyondan aliyoruz.
            steps{
                timeout(time:5, unit:'DAYS'){
                    input message:'Approve terminate'
                }
                sh """
                docker image prune -af
                terraform destroy --auto-approve
                aws ecr delete-repository \
                  --repository-name ${APP_REPO_NAME} \
                  --region ${AWS_REGION} \
                  --force
                """
            }
        }

    }

    post {                      //Pipeline her zaman sonunra yapamasini istedigim seyle. Hata aldigimizda yapacaklari.
        always {
            echo 'Deleting all local images'
            sh 'docker image prune -af'
        }
        failure {

            echo 'Delete the Image Repository on ECR due to the Failure'
            sh """
                aws ecr delete-repository \
                  --repository-name ${APP_REPO_NAME} \
                  --region ${AWS_REGION}\
                  --force
                """
            echo 'Deleting Terraform Stack due to the Failure'
                sh 'terraform destroy --auto-approve'
        }

    }

}
