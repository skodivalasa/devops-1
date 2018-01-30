pipeline {
    agent any

    parameters {
       // string(defaultValue: 'v2', description: '', name: 'buildVersion')
        choice(
            choices: 'Rollingupdate\nBlue-Green',
            description: 'Deployment Type',
            name: 'REQUESTED_ACTION')
    }
    environment {
    // Environment variable identifiers need to be both valid bash variable
    // identifiers and valid Groovy variable identifiers. If you use an invalid
    // identifier, you'll get an error at validation time.
    // Right now, you can't do more complicated Groovy expressions or nesting of
    // other env vars in environment variable values, but that will be possible
    // when https://issues.jenkins-ci.org/browse/JENKINS-41748 is merged and
    // released.
        service = "my-service"
        deployment = "nodejs"
        DEPLOYMENTFILE = "deploy-green.yml"
        VERSION= "${BUILD_ID}"
        image= "pavanraj29/nodejs-app-demo"
     }
    
    stages {
        stage("build") {
            steps {
                echo "${params.buildVersion}"
                sh 'rm -rf HM-Demo'
                sh 'git clone https://github.com/pavaraj29/HM-Demo.git'
            }
        }
        stage("Docker image build") {
            steps {
                sh 'cd HM-Demo &&  sudo docker build -t nodejs-image-new .'
            }
        }
        stage("Docker image tag") {
            steps {
                sh 'sudo  docker tag nodejs-image-new ${image}:${VERSION}'
                //sh 'sudo  docker tag nodejs-image-new ${image}'
            }
        }
        stage("Docker image push") {
            steps {
                sh '''sudo docker login -u pavanraj29 -p Pavan@123
                sudo docker push ${image}:${VERSION}
                cd HM-Demo
                sed -i -e 's/nodejs-app-demo/nodejs-app-demo:'${VERSION}'/g' patch.yaml
                sed -i -e 's/nodejs-app-demo/nodejs-app-demo:'"${VERSION}"'/g' ${DEPLOYMENTFILE}
                sed -i -e 's/nodejs-app-demo:latest/nodejs-app-demo:'${VERSION}'/g' deploy-canary.yaml
                '''
            }
        }
        stage("Image scanning"){
            steps{
                sh '''
                    [ -d "clair-config" ] && sudo rm -rf clair-config
                    sudo mkdir clair-config
                    sudo curl -L https://raw.githubusercontent.com/coreos/clair/master/config.yaml.sample -o $PWD/clair-config/config.yaml 
                    IPADDRESS=`hostname -I | awk '{print $1}'`
                    sudo sed -e "s/localhost/$IPADDRESS/g" -i $PWD/clair-config/config.yaml
                    '''
                sh '''
                    sudo curl -LO https://github.com/optiopay/klar/releases/download/v1.5-RC2/klar-1.5-RC2-linux-amd64
                    sudo chmod +x klar-1.5-RC2-linux-amd64
                    sudo mv klar-1.5-RC2-linux-amd64 /usr/local/bin/klar 
                    CLAIR_ADDR=localhost DOCKER_USER=pavanraj29 DOCKER_PASSWORD=Pavan@123 klar ${image}:${VERSION} || exit 0
                    '''
            }
        }
        stage("Rollingupdate Deployment") {
             when {
                // Only say hello if a "greeting" is requested
                expression { params.REQUESTED_ACTION == 'Rollingupdate' }
            }
            steps {
                sh '''
                cd HM-Demo
                kubectl patch deployment ${deployment} --patch "$(cat patch.yaml)"
                '''
            }
        }
        stage("Blue-green Deployment") {
            when {
                // Only say hello if a "greeting" is requested
                expression { params.REQUESTED_ACTION == 'Blue-Green' }
            }
            steps {
                sh 'kubectl apply -f ${DEPLOYMENTFILE}'
                sh 'kubectl patch svc ${service} -p $"spec:\n selector:\n  - app: nodeapp\n    version: "${VERSION}""'
            }
        }
     }  
}
