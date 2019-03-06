# How to Deploy a Hello World App using Concourse and PCF

This is a simple project to show how to use Concourse and PCF to deploy a Hello World application.
I used the simple Flask app hello world for this, however you can use any language to do this.
You can learn more from [Cloud Foundy](https://www.cloudfoundry.org/). They have examples for a simple
CF push.

![Pipelines](/images/pipeline.png?raw=true "Pipelines")

## Getting Started

You can clone this repo to get this running asap.

### Prerequisites

- [Pivotal](https://account.run.pivotal.io/z/uaa/sign-up) account.
- [Docker](https://www.docker.com/products/docker-engine) installed.
- [Docker-compose](https://docs.docker.com/compose/install/#install-compose) installed.
- [fly CLI](https://concourse-ci.org/download.html) installed.
    Once downloaded you need to add it to your path:
      ```
      sudo mkdir -p /usr/local/bin
      sudo mv ~/Downloads/fly /usr/local/bin
      sudo chmod 0755 /usr/local/bin/fly
      ```

#### Installing

First we want to get our concourse env up and running.

```
docker-compose up -d
```

Then we need to login to concourse, the username & password is admin.

```
fly --target tutorial login --concourse-url http://127.0.0.1:8080 -u admin -p admin
fly --target tutorial sync
```

Great! Now we are logged into concourse, and now we need to build our pipeline. I created
my pipeline to be triggered by my repo, and then build my app and finally cf push.

So now we need to build our pipeline.yml.

First we need to setup our resources. I am using the Docker image
nulldriver/cf-cli-resource. You can read their documentation [here](https://github.com/nulldriver/cf-cli-resource).

```
resource_types:
- name: cf-cli-resource
  type: docker-image
  source:
    repository: nulldriver/cf-cli-resource
    tag: latest

resources:
- name: my-github-repo
  type: git
  source:
    uri: ((my_github_repo))
    branch: master

- name: cf-env
  type: cf-cli-resource
  source:
    api: ((cf_api))
    username: ((cf_login))
    password: ((cf_password))
    org: ((cf_org))
    space: ((cf_space))
    skip_cert_check: true
```
Then we need to create our job.
```
jobs:
- name: hello-world
  public: true
  plan:
  - get: my-github-repo
    trigger: true
  - put: cf-push
    resource: cf-env
    params:
      command: push
      app_name: ((my_app_name))
      memory: 128M
      path: ((my_app_path))
      buildpack: ((cf_buildpack))
```
Ok now that our pipeline.yml is built we close to setting it up, but we first need
to build out our credentials.yml. This way we can feed in the env variables that we need from the
command line. You can do this a few different ways. Check out Stark and Wayne's [turtorial](https://concoursetutorial.com/).

Our credentials.yml should look like this:

```
cf_api: https://api.run.pivotal.io
cf_login: <your username>
cf_password: <your password>
cf_space: <your cf space>
cf_org: <your org name>
cf_buildpack: python_buildpack
my_app_path: my-github-repo/hello_world
my_app_name: <your app name>
my_github_repo: <your github repo uri>
```

To get your cf variables you will need to go to your [pcf console](https://console.run.pivotal.io/).

You will see Orgs and if you have one created you can click into your Org name to
see the spaces available. Plug those in to the credentials.yml and we should be
good to go!

Now that our credentials.yml is setup we can now set the pipelines

```
fly -t tutorial sp -c pipeline.yml -p hello-world -l credentials.yml
fly -t tutorial up -p hello-world
```

So if we navigate back to our dashboard ui http://127.0.0.1:8080/

We will see our job waiting for us to start it. You can click in on the image
and hit the plus button in the top right corner.

Then you will see something like this:

![App Success!](/images/app_success.png?raw=true "App Success!")

And if we navigate to the route listed we will see our app is working!

![](/images/highfive.gif)
