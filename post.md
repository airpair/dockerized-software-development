In a previous blog post, I've described how my development team initializes a rolling deployment to our Amazon hosted Elastic Beanstalk application with a simple commit to GitHub. In that blog post I've described how to build a simple API using the node web framework hapijs and the graph database neo4j, which we have hosted on graph story. By the end of that article, we also created a docker image of the back end service, which is an exact replication of the API which is running on AWS. In this article, we'll proceed to develop a front end application that utilizes the docker image we've created in the previous blog post. We are going to focus on how to utilize docker for software development. But first, I invite you to read [The painful journey of painless deployments](https://www.airpair.com/docker/posts/the-painful-journey-of-painless-deployments) now.

##Initial Docker Set Up
If you are the one who set up the continious deployment system as described in the [previous blog post](https://www.airpair.com/docker/posts/the-painful-journey-of-painless-deployments) or you have docker set up on your system, then you can skip this section as it will briefly describe how to set up Docker on a **Mac OS X**.

###Install Homebrew
Homebrew is the name of a popular package manager for Mac OS X. You can install it via the command line as described on their [homepage](http://brew.sh/):
```bash
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

###Install Virtualbox
Docker is built on top of Linux Kernel technology. Docker on Mac OS X actually runs in a virtual linux environment. Boot2docker is a leightweight linux distribution that will execute docker on a Mac OS X. In order to easily use boot2docker, we're using the free and open source virtualization software virtualbox. Once brew is installed, we can and should use homebrew to get it installed:
```bash
brew install Caskroom/cask/virtualbox
```

###Install Docker and Boot2Docker
Next on our list is boot2docker and docker itself.

Again, we can and should use homebrew to install docker and boot2docker (which is necessary for docker to run on Mac OS X) via brew:
```bash
brew install boot2docker docker
```

In order to use docker, we also have to create a virtual machine once:
```bash
boot2docker init
```

Now you can start boot2docker with `boot2docker start`. Notice, that some environment variables will be printed to the console. It's a good idea to put these into a file that gets loaded once you start your shell. Since I'm using zsh as my preferred shell, I've included these lines in my *~/.zshenv* file:
```
export DOCKER_HOST=tcp://192.168.59.103:2376
export DOCKER_CERT_PATH=/Users/YOUR_USER_NAME/.boot2docker/certs/boot2docker-vm
export DOCKER_TLS_VERIFY=1  
```

##Run your team's docker image
If your team hosts their docker images on [Docker Hub](https://hub.docker.com/) you will have to sign-up for an account in order to get access to your teams' private repositories.

###Authenticate with docker hub
If you're given access to your teams' docker private repositories, you'll have to authenticate with docker hub first. To do so, just run a `docker login` command in the terminal. You will then be prompted to enter your username, password and email address you signed up with at docker hub.

###Pull and run the docker image
We've developed a simple API which is running on our production server as mentioned in the previous blog post. We've also set up our deployment system that a docker image gets pushed to the docker hub registry. Now we want to develop our front end application that consumes that API. But of course we want to avoid querying our production servers, but instead have a local copy of that same API.

On docker hub we found out that the application with the name *epione* has several tags:

- develop.1
- develop.3
- master.2
- master.4

We want to use the *master.4* version of the application, because it matches our most recent production environment.

Hopefully from the application's readme you'll be informed of environment variables that you have to set in order to make the application work as expected with one or multiple *-e* flags.

####Testing the Docker image
Before you start developing, it's a good idea to check that you have the right environment variables, external services (like neo4j servers) are up and running. Refer to your internal documentation to get host names/ip addresses, user names, passwords et cetera.

The resulting command might look similar to this:
```bash
docker run -e EPIONE_TEST_NODE_HOST='0.0.0.0' -e EPIONE_TEST_NODE_PORT='8000' -e EPIONE_TEST_DB_PROTOCOL='https' -e EPIONE_TEST_DB_PORT='7473' -e EPIONE_TEST_DB_HOST='neo-blabla.do-stories.graphstory.com' -e EPIONE_TEST_DB_USER='graphuser' -e EPIONE_TEST_DB_PASS='graphpass' manonthemat/epione:master.4 npm test
```

If our tests pass, as they should, we can start developing.

####Start a Docker container
Besides changing environment variables, we also want to expose the node application's port now. In this case we choose to do an explicit mapping from the container's internal port 8000 to the host's port 8000 using the *-p* flag.

```bash
docker run -e EPIONE_DEVELOPMENT_NODE_HOST='0.0.0.0' -e EPIONE_DEVELOPMENT_NODE_PORT='8000' -e EPIONE_DEVELOPMENT_DB_PROTOCOL='https' -e EPIONE_DEVELOPMENT_DB_PORT='7473' -e EPIONE_DEVELOPMENT_DB_HOST='neo-blabla.do-stories.graphstory.com' -e EPIONE_DEVELOPMENT_DB_USER='graphuser' -e EPIONE_DEVELOPMENT_DB_PASS='graphpass' -p 8000:8000 manonthemat/epione:master.4

```

Notice also how we ommit a custom command at the end of the line. This will default to the defined command from the Dockerfile, which is `npm start`. The application is now serving and can be accessed at 192.168.59.103:8000 via http.

Let's test it:
```bash
$ curl http://192.168.59.103:8000/realusers
[{"row":[{"name":"Matthias Sieber","email":"matthiasksieber@gmail.com"}]}]
```

We are ready to go and can start developing our front end that consumes our API.

##Developing the front end
In this section we will start developing a very simple front end using some components of [Angular Material Design](https://material.angularjs.org). We will consuming the API of our started docker container. This is going to be so simple that we won't even give it a name.

Create a directory for the front end files and in it a `index.html` that can look like this.
```markup,linenums=true
<!DOCTYPE html>
<html ng-app="Poniee">
  <head>
    <link rel="stylesheet" href="https://ajax.googleapis.com/ajax/libs/angular_material/0.8.3/angular-material.min.css" />
    <meta name="viewport" content="initial-scale=1" />
  </head>
  <body layout="column" ng-controller="MainCtrl">
    <md-toolbar layout="row">
      <h1 class="md-toolbar-tools" layout-align-gt-sm="center">Hello Docker</h1>
    </md-toolbar>
    <div layout="row" flex>
      <div>
        <md-radio-group ng-model="usertype">
          <md-radio-button value="real">Real users</md-radio-button>
          <md-radio-button value="fake">Fake users</md-radio-button>
        </md-radio-group>
      </div>
      <div>
        <ul><li ng-repeat="user in users"><a ng-href="mailto:{{user.row[0].email}}">{{user.row[0].name}}</a></li></ul>
      </div>
    </div>
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.3.15/angular.js"></script>
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.3.15/angular-animate.min.js"></script>
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.3.15/angular-aria.min.js"></script>
    <script src="https://ajax.googleapis.com/ajax/libs/angular_material/0.8.3/angular-material.js"></script>
    <script>
      angular.module('Poniee', ['ngMaterial'])
        .constant('API_BASE_URL', 'http://192.168.59.103:8000')
        .controller('MainCtrl', ['$scope', '$http', 'API_BASE_URL', function($scope, $http, API_BASE_URL) {
          $scope.$watch('usertype', function(newVal, oldVal) {
            if (newVal !== oldVal) {
              $http({
                method: 'GET',
                url: API_BASE_URL + '/' + $scope.usertype + 'users'
              }).success(function(data) {
                $scope.users = data;
              });
            }
          });
        }]);
    </script>
  </body>
</html>
```
We have put all of our front end logic in this one file. In this very simple example we can get away with it, because the focus is on how we can use a running docker container, that matches our production environment's API, locally.

In line 34 of the index.html we are setting the path to our *epione* server, which is the docker container that we have [started earlier](#start-a-docker-container). If you're running on a linux or have set a different IP for docker, adjust that value.

From within our front end folder, start a http server to serve the index.html. A simple way to do that is using Python2. Let's start it on port 8080, because 8000 is already taken by the docker container running the back end / API.

    python -m SimpleHTTPServer 8080

Now open up your browser and navigate to http://127.0.0.1:8080.

You should see something that resembles this image.

![No users selected](https://s3.amazonaws.com/docker-dev-article/amfrontend.png)

Let's click the "Real users" radio button to see if our app works and our graphstory neo4j database has data.

![Real users selected](https://s3.amazonaws.com/docker-dev-article/amfrontend_realusers.png)

I've added another fake user to test that our ng-repeat in the index.html works properly:

![Fake users selected](https://s3.amazonaws.com/docker-dev-article/amfrontend_fakeusers.png)

##Conclusion
Not only have we accomplished a [continous integration and deployment](https://www.airpair.com/docker/posts/the-painful-journey-of-painless-deployments) system that pushed working code live to AWS, but we can make the exact same version available for our team members or even the public (without exposing *secrets*) via docker hub.

Feel free to share this article via the social media widget on this page and please leave a review if you have a moment to spare.

Your feedback is much appreciated.
