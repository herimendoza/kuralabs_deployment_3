# **Deployment 3**

##### **Author: Heriberto Mendoza**

##### **Date: 11 October 2022**

---

### **Objective:**

The goal of this deployment is to continue gaining familiarity with the CI/CD pipeline. A simple web application will be deployed across an AWS VPC using Jenkins, gunicorn and nginx.

---

#### ***1. Setting up the Jenkins EC2s***

#### 1a. Jenkins Manager instance creation:

An EC2 instance was created on AWS with certain specification. It was created with an Ubuntu image, and the firewall/securtity group was programmed to allow inbound traffic on ports 8080, 80 and 22. A bash script was written before launching to automatically install Jenkins and the Java Runtime Environment. The following code was used:

```console
#!/bin/bash
sudo apt update
sudo apt install default-jre -y

wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo gpg --dearmor -o /usr/share/keyrings/jenkins.gpg

sudo sh -c 'echo deb [signed-by=/usr/share/keyrings/jenkins.gpg] http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'

sudo apt update
sudo apt install jenkins -y

sudo systemctl start jenkins
```

A relatively small issue was noted: a " \\" (space and backslash) had to be inserted at the end of every line before starting a new line in the user info box during the instance set up step, otherwise the script would not execute properly.

Once Jenkins was active, the following commands were used to install python3-pip (python package manager), python3.10-venv (python virtual environment creator module) and unzip (unzip tool).

NOTE: `sudo apt update && sudo apt upgrade` was run to ensure that all the dependencies were up to date.

```console
$sudo apt install python3-pip -y
$sudo apt install python3.10-venv -y
$sudo apt install unzip -y
```

#### 1b. Jenkins manager VPC:

The Jenkins manager EC2 was created on the default VPC; the subnet, IP address etc were all generated automatically.


#### 1c. Jenkins Agent instance creation + configuration:

A second EC2 was created; this EC2 was designated to be the agent in the infrastructure and what would eventually run the deploy step of the pipeline. The new Jenkins agent was created in an entirely different VPC and different subnet (in the same availability zone though). The subnet in this instaces was a public subnet (VPC routing table had routes for the subnet and internet gateway)

Specifications:
- Ports: 22, 5000
- VPC: CIDR = 172.25.0.0/16, subnet = 172.25.0.0/18
- Installed packages: default-jre, python3-pip, python3.10-venv, nginx

A final step was needed to configure the Jenkins agent: nginx needed to be configured properly in order to host the app.

```console
$ssh -i <key> ubuntu@<agent ec2 ipv4>
$sudo nano /etc/nginx/sites-enabled/default
```

Under server: the default port was changed to 5000:

```console
server {
    listen 5000 default server;
    listen [::]:5000 default server;
    ...
}
```

Under location, the following was added:

```console
location / {
    proxy_pass http://127.0.0.1:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
}
```

Note: In the initial builds, there was an error where the page for the application would not load despite Jenkins returning a successful build. It was later noted that the page wasn't loading because nginx was still listening on the original default port (the port that gunicorn was running on was different). After editing the nginx file to alter the default settings, the nginx service had to be restarted with `$sudo systemctl restart nginx` for the new default settings to take effect. However, now that the port was updated, the builds kept resulting in a 502 bad gateway error. This error was resolved in the 'Deploy' stage.

#### 1d. Connecting Jenkins Manager to Jenkins Agent:

The Jenkins Manager UI was accessed through `<manager ec2 ipv4>:8080` on a web browser and a node was added named 'awsDeploy'. The manager and agent were connected through SSH; the RSA key and the host IP were uploaded to grant the manager access. Since the agent is in a public subnet, the key was all that was needed to establish a connection.

[Jenkins node -- success](https://github.com/herimendoza/kuralabs_deployment_3/blob/8cf742ccae7cee54b94c9038b18c9ad811078035/deploy3_images/jenkins_nodes.png)

#### 1e. Connecting Jenkins Manager to GitHub:

A multibranch pipeline was created to start a build on Jenkins. The link to the appropriate repository (source code and Jenkinsfile) was provided along with the GitHub user credentials. 

#### ***2. The Jenkinsfile***

The Jenkinsfile controls the entire pipeline. There were a total of 5 stages in this pipeline: 'build', 'test', 'clean', 'deploy' and 'post'. They are decribed below: 

#### 2a. The 'Build' stage:

```console
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
```

In the build stage (taking place in the manager EC2) a virtual environment was created, the required dependencies were installed and the application was run.

#### 2b. The 'Test' stage:

```console
stage ('Test') {
    steps {
        sh '''#!/bin/bash
        source test3/bin/activate
        py.test --verbose --junit-xml test-reports/results.xml
        '''
    }
    post{
        always {
            junit 'test-reports/results.xml'
        }
    }
}
```

In the 'Test' stage, the virtual environment was again activated and py.test ran all the files that started with 'test'. In this case, the test was simple string comparison. junit subsequently created a record of the test in the manager's workspace directory, as the 'Test' stage is also taking place in the manager EC2

#### 2c. The 'Clean' stage:

```console
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
```

The agent label specifies that this stage takes place in the Jenkins agent, not the manager. In this stage, a shell script is ran to look for (and terminate if necessary) any gunicorn processes. This is necessary because if a process is running on a specific port (say port 5000), any attempt to run another iteration of said process on the same port would result in a failure because there is already an active process. In this case, running the deployment, making a change to the source code, and then attempting to run the deployment again would not update the app in the web browser. With the 'Clean' stage, any active processes on port 5000 are killed before launching the application again. This solution was demonstrated below: the HTML code was edited to change the header in the application; running the deployment ran the updated application.

[Default application](https://github.com/herimendoza/kuralabs_deployment_3/blob/8cf742ccae7cee54b94c9038b18c9ad811078035/deploy3_images/default_app_page.png)

[Updated application](https://github.com/herimendoza/kuralabs_deployment_3/blob/8cf742ccae7cee54b94c9038b18c9ad811078035/deploy3_images/updated_app_page.png)

#### 2d. The 'Deploy' stage:

```console
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
```

In this stage, a script is run on the agent EC2 to install gunicorn and then run the app on it. gunicorn interacts with the source code to run the application. Since the nginx webserver was configured, any http requests to the correct ip and port are pushed from nginx to gunicorn and gunicorn returns the application. In the above code, the application is meant to be run as a continuous process.

Note: The initial builds, although successful, resulted in a 502 bad gateway error. Further investigation revealed that by default, Jenkins kills all current processes once a stage is completed. In this example, this means that even though Jenkins sucessfully deployed the application on the agent server, once the stage was complete, Jenkins would kill the application. In short, while nginx (the webserver) was still running, gunicorn (the app server) was not, resulting in a 502 error (nginx was pointing to the right port, but gunicorn was not running the app on said port). The solution was a simple Jenkins plugin called 'Pipeline-Keep-Running-Step'. This plugin introduced the use of 'keepRunning{}' wrapping. Anything inside the braces would continue running after the build was complete. Running the gunicorn script in the braces allowed for continuous access to the application. As explained above, the 'Clean' stage before the 'Deploy' stage ensured that in every new build, a new instance of the application was launched, in case there was an update to the code.

[Successful Builds](https://github.com/herimendoza/kuralabs_deployment_3/blob/8cf742ccae7cee54b94c9038b18c9ad811078035/deploy3_images/build_history.png)

#### 2e. The 'Post' stage:

```console
post{
    always{
        emailext to: "heri.mendoza9@gmail.com",
        subject: "jenkins build:${currentBuild.currentResult}: ${env.JOB_NAME}",
        body: "${currentBuild.currentResult}: Job ${env.JOB_NAME}\nMore Info can be found here: ${env.Build_URL}",
        attachLog: true
    }
}
```

In this final stage, the email extension Jenkins plugin was used to send an email notification with the attached build log. In order for this to happen, the email credentials had to be provided along with the smtp server of the email (in this case, google).

#### ***3. Diagram of the Jenkins Pipeline***

[Pipeline](https://github.com/herimendoza/kuralabs_deployment_3/blob/8cf742ccae7cee54b94c9038b18c9ad811078035/deploy3_images/Deploy3_pipeline.png)

#### ***4. VPC Architecture***

[VPC](https://github.com/herimendoza/kuralabs_deployment_3/blob/8cf742ccae7cee54b94c9038b18c9ad811078035/deploy3_images/Deploy3_architecture.png)

#### ***5. Type of stack***

An overview of the pipeline and VPC architecture highlights some of the important tools that are used in this "stack" of software and tools for deployment. On the front end, HTML, CSS and Javascript were used to render the web pages. Behind that was nginx which redirected traffic to a gunicorn instance running a python app, all on Linux. There were no databases involved in this application. This stack most closely resembles a simplified version of an LEMP stack (a stack that uses Linux, Nginx, MySQL and PHP).


#### ***6. Improvements***

In thinking about how to improve this deployment:
- It would merit consideration to have one VPC with EC2's in private subnets in different availablity zones, instead of two VPCs with public subnets. This would mitigate security risks that come with not securing subnets. In a real world, enterprise level environment, all the servers would ideally be in private subnets with reduntant servers in other AZs. The user would access the app through a bastion host.
- More tests (using cypress) could be added to this pipeline. A third EC2 could be instantiated with Cypress and a Cypress test stage could be added to the pipeline. The application would be put through tests on the third server before being deployed on the deployment server. 
