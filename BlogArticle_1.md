Back in December of 2022 I attended the ServerSide.Swift-Conference in London and while listening and watching to the different keynotes, I realised that there still is one problem. Deploying a Vapor app.
I get the point, with an iOS/Macos app it is pretty simple. Just send it to Apple and they will hanlde it. That is not possible for Vapor Apps.
So what do we do now?
The answer is simple, search for a tutorial on the internet, or follow this tutorial, where I will cover  everything you will need.
First things first, we need some kind of binary, which we can execute. This part is simple, since the wonderful guys at vapor already included a docker image, that will basically do that.

This tutorial started out as a simple how-to, but grew a lot over the time of writting. This tutorial will be a series of 3 parts.
This first part will cover the basics and a light intoduction to github actions and docker.
The second part will cover all kinds of optimisations for the build process.
the third part will cover the deployment to different services

The first part will cover the following steps:
- Create a Docker-image that can be deployed "anywhere"
- Have the CI do the heavy lifitng for us

Let's start with the docker image and have github create it for us.
Ingredients:
- Github account
- that is it

I will show you a very easy way to use Github Actions to create and host the Docker image for you. Github will help us with Github-Actions to create our App as a Docker-Image and host the docker image on it's own Github Container Registry (ghcr.io)
First off we need to create a Github-action.
### Creating a Github action
In your repository root folder, go into or create the folders `.github/workflows`.
This will be our home for all the Github Actions we can think of.
A Github Action is described as `*.yml` file. Let's create and open the file `build-and-push.yml` 
We will start simple and only use a few lines at the beginning.

To understand the whole thing a little bit better, imagine you have a Computer(`runner`), where every tool you need is already installed and you can just start. So there are 4 basic steps we have to do:
1. checkout the branch/repository
2. build the docker image
3. login to the Container Registry, this could be hub.docker.com or Githubs equivalent (ghcr.io)
4. push docker image to container registry

```yaml 
name: Build and Push # 1
on: push # 2

jobs:
  build:
    runs-on: ubuntu-latest # 3
    steps:
    - uses: actions/checkout@v3 # 4

    - name: Build image # 5
      run: docker build --tag ghcr.io/${{ github.actor }}/${{ github.event.repository.name }}:latest . # 6

    - name: Login to GitHub Container Registry
      uses: docker/login-action@v2 # 7
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Push image to container registry # 8
      run: docker push ghcr.io/${{ github.actor }}/${{ github.event.repository.name }}:latest

```

Let's go through the definition together.
1. `name` defines the name of this action and will be visible on github
2. `on` is the trigger of this action. There are a lot and more precise ones than `push` have a look at the very good [documentation](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows).
3. `runs-on` defines the operating system (OS) of the `runner`. You have the option to choose between 3 (`ubuntu-latest`, `windows-latest`, or `macos-latest`). But here is the important part. Each OS has different `Minute multiplier`

| OS | Minute multiplier |
|----:|:----:|
| Linux | 1 |
| Windows | 2 |
| macOS | 10 |

I would avoid using macOS runners as much as possible, but sometimes you just can't get around it like when you are building for iOS or macOS, but vapor will run on linux.

4. `uses` will let the runner know, that there is a predefined action that is being used. In this case we use the checkout action to check out the repository. This is most likely the first step, since the runner does not know anything about your code.
5. `name` here is an optional value that you can define as you want, it will be visible on github.
6. `run` is the syntax to run a bash command you could also start the first line with `|` to write a multiline command in the next line. Very useful if you know the bash commands to execute. Here we run the docker build command. Github actions also has variables that can be set and read from. `${{ variable_name }}` is the syntax for those variables. Github already predefined a lot of variables for us to use, such as the actor of this repository (owner) or the name of the repository. Github has a lot of these, but sadly not one single place where you can find them all, so go ahead and look in the documentation mentioned before.
7. Another predefined action, but this time is has an optinal name and additional parameters listed under `with` these can be found in the documentation of the action itself.
8. last step in our github action is finally pushing the ready-to-go docker-image to ghcr.io

Now you are all set to use the docker-image for where ever you want.
You might have to login to github on the publishing service to have access to a private docker-image.

This approach is straight forward and easy to follow, but can be optimized.
The upsides:
- very easy to understand and use
- can be used everywhere

The downsides:
- always spins up a docker container for building the app
- it will take a longer time (every run will take up to 7 min and this is without any additional dependencies)
- will not use previous build-artifacts to improve build time
- can not access private dependencies (at least not that easy)

