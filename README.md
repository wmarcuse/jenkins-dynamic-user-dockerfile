# jenkins-dynamic-user-dockerfile

This example creates a defined user in the pipeline agent Docker container, using the outerscope GID and UID from the jenkins server user.

With this configuration you can avoid `Permission Denied` errors when using `pip` inside pipeline agent containers with Python.


### Jenkinsfile

    #!groovy
    pipeline {
    
        environment {
                JENKINS_USER_NAME = "${sh(script:'id -un', returnStdout: true).trim()}"
                JENKINS_USER_ID = "${sh(script:'id -u', returnStdout: true).trim()}"
                JENKINS_GROUP_ID = "${sh(script:'id -g', returnStdout: true).trim()}"
        }
        agent {
            dockerfile {
                filename 'Dockerfile.build'
                additionalBuildArgs '''\
                --build-arg GID=$JENKINS_GROUP_ID \
                --build-arg UID=$JENKINS_USER_ID \
                --build-arg UNAME=$JENKINS_USER_NAME \
                '''
            }
        }
    
        stages {
            stage('Build Dependencies') {
                steps {
                    sh "pip install -r requirements.txt"
                }
            }
    }
        post {
            always {
                sh "Job finished"
            }
        }
    }
    
### Dockerfile.build

    FROM python:3.8.3
    
    ARG GID
    ARG UID
    ARG UNAME
    
    ENV GROUP_ID=$GID
    ENV USER_ID=$UID
    ENV USERNAME=$UNAME
    
    RUN mkdir /home/$USERNAME
    
    RUN groupadd -g $GROUP_ID $USERNAME
    RUN useradd -r -u $USER_ID -g $USERNAME -d /home/$USERNAME $USERNAME
    RUN chown $USERNAME:$USERNAME /home/$USERNAME
    
    USER $USERNAME
    WORKDIR /home/$USERNAME
    
    RUN python -m venv testpipe_venv
    
    CMD ["/bin/bash -c 'source testpipe_venv/bin/activate'"]
