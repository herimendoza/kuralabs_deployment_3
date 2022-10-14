pipeline {
  agent any
   stages {
    stage ('Build') {
      steps {
        sh '''#!/bin/bash
        python3 -m venv test3
        source test3/bin/activate
        pip install pip --upgrade
        pip install -r requirements.txt
        export FLASK_APP=application
        flask run &
        '''
     }
   }
    stage ('Test') {
      steps {
        sh '''#!/bin/bash
        source test3/bin/activate
        py.test --verbose --junit-xml test-reports/results.xml
        echo 'TESTING WEBHOOK + EMAIL NOTIF'
        ''' 
      }
      post{
        always {
          junit 'test-reports/results.xml'
        }
      }
    }
     stage ('Clean') {
       agent{label 'awsDeploy'}
       steps {
         sh '''#!/bin/bash
         if [[ $(ps aux | grep -i "gunicorn" | tr -s " " | head -n 1 | cut -d " " -f 2) != 0 ]]
         then
          ps aux | grep -i "gunicorn" | tr -s " " | head -n 1 | cut -d " " -f 2 > pid.txt
          kill $(cat pid.txt)
          exit 0
          fi
          '''
       }
     }
     stage ('Deploy') {
       agent{label 'awsDeploy'}
       steps {
         keepRunning {
           sh '''#!/bin/bash
           pip install -r requirements.txt
           pip install gunicorn
           python3 -m gunicorn -w 4 application:app -b 0.0.0.0 --daemon
           '''
         }
       }
     }
   
  }
  post{
    always{
      emailext to: "heri.mendoza9@gmail.com",
      subject: "jenkins build:${currentBuild.currentResult}: ${env.JOB_NAME}",
      body: "${currentBuild.currentResult}: Job ${env.JOB_NAME}\nMore Info can be found here: ${env.BUILD_URL}",
      attachLog: true
    }
  }
 }

