I went to the ServerSide.Swift-Conference in London and survived to tell the tail.
This is one article of a series of articles about my experiences in London. Some might be more Technical, some might just be impressions. I hope You enjoy them.

TLDR:
Optimize github actions to create and push a swift vapor app to the github Container Registry.
[Here you can just jump to the implementation](https://github.com/elmoritz/deploy-vapor-app/tree/stage/github-action-optimized)



Last time we created a github action to post our vapor project to the github container registry. Today we will optimize the process so it will not take as long or cost as much money.

This is part 2
The first part covered the basics and a light intoduction to github actions and docker.
This second part will cover all kinds of optimisations for the build process.
The third part will cover the deployment to different services

The second part will cover the following steps:
- Optimize the github action created in part 1
- use artifacts and caching to speed up the process

While using a docker image to build the project is very nice, it has a few short-comings.
- it will always be a clean build. no incremental builds possible.
- there for it will always have to download all dependecies from scratch.
- you can not use dependencies from a private repository (you can try it, but it takes a lot more knowledge than I want to acquire)

With a few easy steps we can optimize a lot. Let's start with the multistage docker image.
When we look into the docker file, you can see that there are actually 2 different images defined.
```
FROM swift:5.7-jammy as build
```
AND
```
FROM ubuntu:jammy
```
The first one is used to build the project and the second one is the one you will actually use.
As I found out a few weeks ago, Github includes the swift library on `ubuntu-latest`, so lets use it.
Our goal is to only have one image defined in the `Dockerfile`  and get rid of the `as build` part.
Let's duplicate the existing `Dockerfile` and rename it to `Dockerfile.multiStage`, that way we still have the original version. We are going to change the `Dockerfile`. Remove everything  above the line
```
FROM ubuntu:jammy
```
The `jammy` part might differ in your case. Also remove the line 
```
&& apt-get -q dist-upgrade -y \
```
because we do not want to update the distribution of ubuntu (if there is a new version this will take a long time)
One last change we have to do here is to remove `--from=build` from the following line
```
COPY --from=build --chown=vapor:vapor /staging /app
```

because we don't use the `build` part anymore. Now you should have a nice clean and simple Dockerfile.
Since we removed the build phase from the Dockerfile, we have to do this part manually before.

Now we have to change the Github Action to build the project, before we want to move it from `/staging` to `/app` 

For the new `Dockerfile` to work we need to build the app and move it into `/staging`. One small thing the original Dockerfile does is collect all Resources from your app and every dependency and moves it to the `/staging` folder as well. So let's get started.

In `build-and-push.yml` after the Checkout, we will add our new build step and add one step to do the Resources "magic"




instead of only calling `docker build` we need to do
```
- name: Build
  run: swift build -c release -Xswiftc -O --static-swift-stdlib

- name: Create staging directory
- run: |
  mkdir staging
  cp "$(swift build -c release --show-bin-path)/Run" ./staging
  find -L "$(swift build -c release --show-bin-path)/" -regex '.*\.resources$' -exec cp -Ra {} ./staging/ \;
  [ -d ./Public ] && { mv ./Public ./staging/Public && chmod -R a-w ./staging/Public; } || true
  [ -d ./Resources ] && { mv ./Resources ./staging/Resources && chmod -R a-w ./staging/Resources; } || true

```

| github action | 1. run | 2. run | 3. run |
|--:|:-:|:-:|:-:|
| plain | 386s | 406s | 390s |
| optimized | 384s | 273s | 267s |

As you can see in the table the optimized github action takes just as much time as the unoptimized action on its first run, but every subsequent run, where the `Package.resolved` file is not changed, will actually be a lot faster.
