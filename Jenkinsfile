pipeline {
    agent any

    parameters {
       // string(defaultValue: 'v2', description: '', name: 'buildVersion')
        choice(
            choices: 'Rollingupdate\nBlue-Green\nCanary',
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
        VERSION= "${BUILD_ID}"
        image= "pavanraj29/nodejs-app"
     }
    
    stages {
        stage("build") {
            steps {
                echo "${params.buildVersion}"
                sh 'sudo rm -rf node-js-sample'
                sh 'git clone https://github.com/pavaraj29/node-js-sample.git'
            }
        }
        stage("Docker image build") {
            steps {
                sh 'cd node-js-sample &&  sudo docker build -t nodejs-image-new .'
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
                sed -i -e 's/nodejs-app:latest/nodejs-app:'${VERSION}'/g' deploy-canary.yaml
                '''
            }
        }
        stage("Rollingupdate Deployment") {
             when {
                // Only say hello if a "greeting" is requested
                expression { params.REQUESTED_ACTION == 'Rollingupdate' }
            }
            steps {
                sh 'echo Hello'
                sh 'kubectl patch deployment ${deployment} -p $"spec:\n   containers:\n   - name: front-end\n     image: ${image}"'
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
        stage("Canary Deployment") {
            when {
                // Only say hello if a "greeting" is requested
                expression { params.REQUESTED_ACTION == 'Canary' }
            }
            steps {
                sh '''
                        kubectl apply -f deploy-canary.yaml
                        sleep 60
                        SERVICE_IP=`kubectl get svc my-service -o jsonpath="{.status.loadBalancer.ingress[0].*}"`
                        for i in `seq 1 10`; do curl http://$SERVICE_IP/>>tmpfile; sleep 1;  done
                        if grep -q "Hello World-v4" "tmpfile"; then 
                        kubectl set image deployment/nodejs front-end=${image}:latest
                        fi
                        #rm tmpfile
                '''
            }
        }
     }  
}
