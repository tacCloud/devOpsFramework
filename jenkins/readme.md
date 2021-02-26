# This comes from: https://github.com/marcel-dempers/docker-development-youtube-series/tree/master/jenkins

# Jenkins on Amazon Kubernetes

For running Jenkins on AMAZON, start [here](./amazon-eks/readme.md)

# Jenkins on Local (Docker Windows \ Minikube \ etc)

For running Jenkins on Local Docker for Windows or Minikube <br/>
Watch the [video](https://youtu.be/eRWIJGF3Y2g)

# Setting up Jenkins Agent

After installing `kubernetes-plugin` for Jenkins
* Go to Manage Jenkins | Bottom of Page | Cloud | Kubernetes (Add kubenretes cloud)
* Fill out plugin values
    * Name: kubernetes
    * Kubernetes URL: https://kubernetes.default:443
    * Kubernetes Namespace: jenkins
    * Credentials | Add | Jenkins (Choose Kubernetes service account option & Global + Save)
    * Test Connection | Should be successful! If not, check RBAC permissions and fix it!
    * Jenkins URL: http://jenkins
    * Tunnel : jenkins:50000
    * Apply cap only on alive pods : yes!
    * Add Kubernetes Pod Template
        * Name: jenkins-slave
        * Namespace: jenkins
        * Service Account: jenkins
        * Labels: jenkins-slave (you will need to use this label on all jobs)
        * Containers | Add Template
            * Name: jnlp
            * Docker Image: rmccabe3701/jenkins-slave
            * Command to run : <Make this blank>
            * Arguments to pass to the command: <Make this blank>
            * Allocate pseudo-TTY: yes
            * Add Volume
                * HostPath type
                * HostPath: /var/run/docker.sock
                * Mount Path: /var/run/docker.sock
        * Timeout in seconds for Jenkins connection: 300
* Save

# Test a build

To run docker commands inside a jenkins agent you will need a custom jenkins agent with docker-in-docker working.
Take a look and build the docker file in `./dockerfiles/jenkins-agent`
Push it to a registry and use it instead of above configured `* Docker Image: jenkins/jnlp-slave`
If you do not use the custom image, the below pipeline will not work because default `* Docker Image: jenkins/jnlp-slave` public image does not have docker ability.

* Add a Jenkins Pipeline

```
node('jenkins-slave') {

     stage('unit-tests') {
        sh(script: """
            docker run --rm alpine /bin/sh -c "echo hello world"
        """)
    }
}
```


ISSUE:
 * Cannot run golang in the master jenkins pod (something about Alpine not being able to run glibc programs)


HOWTO:

* Connect an "always-on" Jenkins slave (Connect to docker-slave via ssh)

```bash
#Launch Jenkins via Docker (not sure how to connect to docker image from k8s)
#NOTE: cannot launch docker containers directly from Jenkins as it doesn't run as root :(
docker run -i --rm --name jenkins-master -v /Users/rjmccabe/repos/k8s_demo/jenkins_storage:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -p 8080:8080 -p 50000 \
  jenkins/jenkins:2.235.1-lts-alpine

#Launch the slave (this runs sshd)
docker run -i --rm --name jenkins-slave-docker \
  -v /var/run/docker.sock:/var/run/docker.sock \
   bibinwilson/jenkins-slave:latest

Then add the node in the Jenkins WebUi
```

* Add a dynamic worker (that does golint, gotest, builds image and pushes)

cd dockerfiles
docker build -t rmccabe3701/jenkins-slave .
docker push rmccabe3701/jenkins-slave


docker run -i --rm --name jenkins-slave-docker \
  -v /var/run/docker.sock:/var/run/docker.sock \
   rmccabe3701/jenkins-slave:latest


Set DOCKER_USERNAME and DOCKER_PASSWORD as credentials (secret text) in Jenkins WebUI


Interesting webapp (and fluentd logging):
https://github.com/chrisvugrinec/aks-logging/tree/master/got-webapp

TODO:

 * Figure out how to connect a docker Jenkins slave:
 docker run -i --rm --name agent --init jenkins/agent java -jar /usr/share/jenkins/agent.jar
  (not sure how to connect with a Docker container from a kubernetes pod)

#NOTE: cannot launch docker containers directly from Jenkins as it doesn't run as root :(
 docker run -i --rm --name jenkins-master -v /Users/rjmccabe/repos/k8s_demo/jenkins_storage:/var/jenkins_home \
   -v /var/run/docker.sock:/var/run/docker.sock \
   -p 8080:8080 -p 50000 \
   jenkins/jenkins:2.235.1-lts-alpine

 docker run -i --rm --name agent \
   -v /var/run/docker.sock:/var/run/docker.sock \
   --init jenkins/agent java -jar /usr/share/jenkins/agent.jar

 docker run -i --rm --name agent \
   -v /var/run/docker.sock:/var/run/docker.sock \
   --init jenkins/agent java -jar /usr/share/jenkins/agent.jar

 docker run -i --rm --name jenkins-slave-docker \
   -v /var/run/docker.sock:/var/run/docker.sock \
    bibinwilson/jenkins-slave:latest

 * Run some golang lint/tests in the pipeline

 * build the docker image and push to dockerhub (credentials?)
