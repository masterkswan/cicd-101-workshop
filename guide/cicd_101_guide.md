# Intro to CI/CD Workshop

This document will walk through implementing a simple CI/CD pipeline into a codebase using [CircleCI](https://circleci.com/)
The following will be demonstrated:

- Integrating CircleCI with a Github project
- A unittest for a python flask application
- Implementing a CI/CD pipeline in the codebase using a CircleCI config file in the project
- Build a Docker image
- Deploy the Docker image to [Docker Hub](https://hub.docker.com)

## Prerequisites

Before you get started you'll need to have these things:

- [Github Account](https://github.com/join)
- [CircleCI](https://circleci.com/signup/) account
- Docker Hub account](https://hub.docker.com)
- Fork then clone the [cicd-101-workshop repo](https://github.com/ariv3ra/cicd-101-workshop) locally

<!--  
 - Set [Project Environment Variables](https://circleci.com/docs/2.0/env-vars/#setting-an-environment-variable-in-a-project) which specify your Docker Hub **Username** and **Password**in the CircleCI dashboard
 - SSH Access to a cloud server. You can [add a SSH key to your account](https://circleci.com/docs/2.0/add-ssh-key/) via the CircleCI portal. For this post I'll be using a [Digital Ocean](https://www.digitalocean.com) server but you can use whatever server/cloud provider you desire
 - Create a deployment script on the host server that will be used to deploy this application [here is an example deployment script deploy_app.sh](https://raw.githubusercontent.com/ariv3ra/python-circleci-docker/master/deploy_app.sh)
 -->

 After you have all the pre-requisites complete you're ready to proceed to the next section.

## CircleCI: Add Project

In order for the CircleCI platform to integrate with projects it must have access to your codebase. In this section demonstrate how to give CircleCI access to a project on Github via the CircleCi dashboard.

- Login to the [dashboard](http://circleci.com/vcs-authorize/)
- Click the **Add Project** icon on the left menu
- Find the `cicd-101-workshop` project in the list
- Click the corresponding **Set Up Project** button on the right 

![Add Project](./images/cci_add_project.png)

-

<!-- Left off here --> 

# The App

For this post I'll be using a simple python [Flask](http://flask.pocoo.org/) and you can find the complete [source code for this project here](https://github.com/ariv3ra/python-circleci-docker) and you can `git clone` it locally. The app is a simple webserver that renders html when a request is made to it. The flask application lives in the `hello_world.py` file:

```
from flask import Flask

app = Flask(__name__)

def wrap_html(message):
    html = """
        <html>
        <body>
            <div style='font-size:120px;'>
            <center>
                <image height="200" width="800" src="https://infosiftr.com/wp-content/uploads/2018/01/unnamed-2.png">
                <br>
                {0}<br>
            </center>
            </div>
        </body>
        </html>""".format(message)
    return html

@app.route('/')
def hello_world():
    message = 'Hello DockerCon 2018!'
    html = wrap_html(message)
    return html

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

The key take away in this code the `message` variable within the `hello_world()` function.  This variable specifies a string value and the value of this variable will be tested for a match in a unittest.

# Testing Code

All code must be tested to ensure that quality stable code is being released to the public.  Python comes with a testing framework named [unittest](https://docs.python.org/2/library/unittest.html) and I'll be using that for this post.  We now have a complete flask application and it needs a companion unittest that will test the application and ensure it's functioning as designed. The unittest file `test_hello_world.py` is the unittest for our hello_world.py app and I'll walk your through the code.

```
import hello_world
import unittest

class TestHelloWorld(unittest.TestCase):

    def setUp(self):
        self.app = hello_world.app.test_client()
        self.app.testing = True

    def test_status_code(self):
        response = self.app.get('/')
        self.assertEqual(response.status_code, 200)
    
    def test_message(self):
        response = self.app.get('/')
        message = hello_world.wrap_html('Hello DockerCon 2018!')
        self.assertEqual(response.data, message)

if __name__ == '__main__':
    unittest.main()
```

```
import hello_world
import unittest
```

Import the `hello_world` application using the `import` statement which gives the test access to the code in the `hello_world.py`. Next import the `unittest` modules and start defining test coverage for the application.

`class TestHelloWorld(unittest.TestCase):`
The TestHelloWorld is instantiated from the base class `unittest.Test` which is the smallest unit of testing. It checks for a specific response to a particular set of inputs. unittest provides a base class, TestCase, which may be used to create new test cases.

```
def setUp(self):
        self.app = hello_world.app.test_client()
        self.app.testing = True
```

`setUp()` is a class level method called to prepare the test fixture. This is called immediately before calling the test method. In this example we create define a variable named `app` and instantiate it as `app.test_client()` 
object from the hello_world.py code.

```
def test_status_code(self):
    response = self.app.get('/')
    self.assertEqual(response.status_code, 200)
```

`test_status_code()` is a method and it specifies an actual test case in code.  This test case makes a `get` request to the flask application and captures the app's response in the `response` variable. The `self.assertEqual(response.status_code, 200)` compares the value of the `response.status_code` result to the expected value of `200` which signifies the `get` request was successful. If the server responds with a status_code other that 200 the test will fail.

```
def test_message(self):
    response = self.app.get('/')
    message = hello_world.wrap_html('Hello DockerCon 2018!')
    self.assertEqual(response.data, message)
```

`test_message()` is another method that specifies a different test case.  This test case is designed to check the value of the `message` variable that is defined in the `hello_world()` method from the hello_world.py code. Like the previous test a **get** call is made to the app and the results are captured in a `response` variable. The following line :

```
message = hello_world.wrap_html('Hello DockerCon 2018!')
```

The `message` variable is assigned the resulting html from the `hello_world.wrap_html()` helper method which is defined in the hello_world app.  The string `Hello DockerCon 2018` is supplied to the `wrap_html()` method which is then injected & returned in html. The `test_message()` will verify that the message variable in the app will match the expected string in this test case. If the strings don't match then the test will fail.

# CI/CD Pipelines

Now that we're clear on the application and it's unit tests it time to implement a Continuos Integration / Continuous Deployment pipeline into the codebase. Implementing a CI/CD pipeline using CircleCI is very simple. Before continuing make sure you do the following:

- [Create a CircleCI Account](https://circleci.com/signup/)
- [Setup your build in CircleCI](https://circleci.com/docs/2.0/#setting-up-your-build-on-circleci)

## Implement a CI/CD pipeline

Once your project is setup in the CircleCI platform any commits pushed upstream will be detect and CircleCI will execute the job defined in your `config.yml` file.

You will need to create a new directory in the repo's root and a yaml file within this new directory. The new assets must follow these naming schema - directory: `.circleci/` file: `config.yml` in your project's git repository. This directory and file basically define your CI/CD pipeline adn configuration for the CircleCI platform.

## config.yml File

The config.yml is where all of the CI/CD magic happens. Below is an example of the file used in the example file and I'll briefly explain what's going on within the syntax:

```
version: 2
jobs:
  build:
    docker:
      - image: circleci/python:2.7.14
        environment:
          FLASK_CONFIG: testing
    steps:
      - checkout
      - run:
          name: Setup VirtualEnv
          command: |
            echo 'export TAG=0.1.${CIRCLE_BUILD_NUM}' >> $BASH_ENV
            echo 'export IMAGE_NAME=python-circleci-docker' >> $BASH_ENV 
            virtualenv helloworld
            . helloworld/bin/activate
            pip install --no-cache-dir -r requirements.txt
      - run:
          name: Run Tests
          command: |
            . helloworld/bin/activate
            python test_hello_world.py
      - setup_remote_docker:
          docker_layer_caching: true
      - run:
          name: Build and push Docker image
          command: |
            . helloworld/bin/activate
            pyinstaller -F hello_world.py
            docker build -t ariv3ra/$IMAGE_NAME:$TAG .
            echo $DOCKER_PWD | docker login -u $DOCKER_LOGIN --password-stdin
            docker push ariv3ra/$IMAGE_NAME:$TAG
      - run:
          name: Deploy app to Digital Ocean Server via Docker
          command: |
            ssh -o StrictHostKeyChecking=no root@hello.dpunks.org "/bin/bash ./deploy_app.sh ariv3ra/$IMAGE_NAME:$TAG"
```

The `jobs:` key represents a list of jobs that will be run.  A job encapsulates the actions to be executed. If you only have one job to run then you must give it a key name `build:` you can get more details about [jobs and builds here](https://circleci.com/docs/2.0/configuration-reference/#jobs)

The `build:` key is composed of a few elements:

- docker:
- steps:

The `docker:` key tells CircleCI to use a [docker executor](https://circleci.com/docs/2.0/configuration-reference/#docker) which means our build will be executed using docker containers.

`image: circleci/python:2.7.14` specifies the docker image that the build must use

### steps: 

The `steps:` key is a collection that specifies all of the commands that will be executed in this build. The first action that happens the `- checkout` command that basically performs a git clone of your code into the build environment.

The `- run:` keys specify commands to execute within the build.  Run keys have a `name:` parameter where you can label a grouping of commands. For example `name: Run Tests` groups the test related actions which helps organize and display build data within the CircleCI dashboard.

**Important NOTE:** Each `run` block is equivalent to separate/individual shells or terminals so commands that are configured or executed will not persist in latter run blocks. Use the `$BASH_ENV` work around in the [Tips & Tricks section](https://circleci.com/docs/2.0/migration/#tips-for-setting-up-circleci-20)

```
- run:
    name: Setup VirtualEnv
    command: |
      echo 'export TAG=0.1.${CIRCLE_BUILD_NUM}' >> $BASH_ENV
      echo 'export IMAGE_NAME=python-circleci-docker' >> $BASH_ENV 
      virtualenv helloworld
      . helloworld/bin/activate
      pip install --no-cache-dir -r requirements.txt
```

The `command:` key for this run block has a list of commands to execute. These commands set the `$TAG` & `IMAGE_NAME` custom environment variables that will be used throughout this build. The remaining commands set up the [python virtualenv](https://virtualenv.pypa.io/en/stable/) & installs the python dependencies specified in the `requirements.txt` file.

```
- run:
    name: Run Tests
    command: |
      . helloworld/bin/activate
      python test_hello_world.py
```            

In this run block the command executes tests on our application and if these tests fail the entire build will fail and will require the developers to fix their code and recommit.

```
- setup_remote_docker:
    docker_layer_caching: true
```

This run block specifies the [setup_remote_docker:](https://circleci.com/docs/2.0/glossary/#remote-docker) key which is a feature that enables building, running and pushing images to Docker registries from within a Docker executor job. When docker_layer_caching is set to true, CircleCI will try to reuse Docker Images (layers) built during a previous job or workflow. That is, every layer you built in a previous job will be accessible in the remote environment. However, in some cases your job may run in a clean environment, even if the configuration specifies docker_layer_caching: true.

Since we're building a Docker image for our app and pushing that image to Docker Hub the `setup_remote_docker:` is required.

```
- run:
    name: Build and push Docker image
    command: |
      . helloworld/bin/activate
      pyinstaller -F hello_world.py
      docker build -t ariv3ra/$IMAGE_NAME:$TAG .
      echo $DOCKER_PWD | docker login -u $DOCKER_LOGIN --password-stdin
      docker push ariv3ra/$IMAGE_NAME:$TAG
```

The **Build and push Docker image** run block specifies the commands that package the application into a single binary using pyinstaller then continues on to the Docker image building process.

```
docker build -t ariv3ra/$IMAGE_NAME:$TAG .
echo $DOCKER_PWD | docker login -u $DOCKER_LOGIN --password-stdin
docker push ariv3ra/$IMAGE_NAME:$TAG
```

These commands build the docker image based on the `Dockerfile` included in the repo. [Dockerfile](https://docs.docker.com/engine/reference/builder/) is the instruction on how to build the Docker image.

```
FROM python:2.7.14

RUN mkdir /opt/hello_word/
WORKDIR /opt/hello_word/

COPY requirements.txt .
COPY dist/hello_world /opt/hello_word/

EXPOSE 80

CMD [ "./hello_world" ]
```

The `echo $DOCKER_PWD | docker login -u $DOCKER_LOGIN --password-stdin` command uses the $DOCKER_LOGIN and $DOCKER_PWD env variables set in the CircleCI dashboard as credentials to login & push this image to Docker Hub.

```
- run:
    name: Deploy app to Digital Ocean Server via Docker
    command: |
      ssh -o StrictHostKeyChecking=no root@hello.dpunks.org "/bin/bash ./deploy_app.sh ariv3ra/$IMAGE_NAME:$TAG"
```            

The final run bock deploys our new code to a live server running on the Digital Ocean platform. **Make sure** that you've created a deploy script on the remote server.  The ssh command access the remote server and executes the `deploy_app.sh` script on the the server and specifies: **ariv3ra/$IMAGE_NAME:$TAG** which specifies the image to pull & deploy from Docker Hub.

After the job successfully completes, the new application should be running on the target server you specified in you config.yml.

# Summary

In review this post should guide you through implementing a CI/CD pipeline into your code.  Though this example is built using python technologies the general build, test and deployment concepts can easily be implemented in whatever language or framework you desire. The examples in this post are also simple but you can expand on them and tailor them to your pipelines. CircleCI has great [documentation](https://circleci.com/docs/2.0/) so don't hesitate to research our docs site and if you really get stuck you can also reach out to the CircleCI community via the [https://discuss.circleci.com/](https://discuss.circleci.com/) community/forum site.


