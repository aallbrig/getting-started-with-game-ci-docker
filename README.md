## Getting Started with Game CI Docker Images
**Table of Contents**
- Background
- Important Resources
- Activate a License
- Create a new Unity Project
- Test Runner
- Builder
### Background
Hello! I'm writing this guide because [game.ci](https://game.ci/) had a big impact on my hobbyist game developer career. I see this guide as a way I can contribute back to the community whose technology has had a big impact on me.

When I started learning how to do game development one thing I decided to do was to also understand automated testing and continuous integration & continuous delivery (CI/CD). Game.ci had done the heavy lifting required to containerize and operationalize unity into containers. If you are a creator, a maintainer, or have helped contribute in any way to the game.ci project in any way I humbly thank you. I stand on the shoulders of giants.

### Important Resources
[Game.ci](https://game.ci/) is the website that contains the documentation on how to use this technology. The easiest way of using game.ci is inside what I call "CI/CD cloud runners" which just means as GitHub Actions, GitLab CI, or even circle CI. Using the docker containers directly is an exercise left up for the user. At the time of writing my understanding is that a CLI will be used to unlock game.ci CI/CD activities locally.

The [game-ci/docker](https://github.com/game-ci/docker) repo contains definitions for the docker images eventually used by the CI/CD cloud runners. You can see a [base image](https://github.com/game-ci/docker/tree/main/images/ubuntu/base), an [image for unity-hub](https://github.com/game-ci/docker/tree/main/images/ubuntu/hub), and another [image for unity-editor](https://github.com/game-ci/docker/tree/main/images/ubuntu/editor). Each operating system (macOS, windows, ubuntu/linux) has this set of three.

These images are published up to [unityci](https://hub.docker.com/u/unityci), which is game.ci's docker hub. If you browse, say, the [unityci/editor tags](https://hub.docker.com/r/unityci/editor/tags) you will find many unity editor versions preloaded with items required to build

Important for reverse engineering the docker usage is the **dist** folder found in each repo that corresponds to a CI/CD cloud runner action.
- [unity-request-activation-file/tree/main/dist](https://github.com/game-ci/unity-request-activation-file/tree/main/dist)
- [unity-activate/tree/main/dist](https://github.com/game-ci/unity-activate/tree/main/dist)
- [unity-test-runner/tree/main/dist](https://github.com/game-ci/unity-test-runner/tree/main/dist)
- [game-ci/unity-builder/tree/main/dist](https://github.com/game-ci/unity-builder/tree/main/dist)
- [game-ci/unity-return-license](https://github.com/game-ci/unity-return-license)

## Before Getting Started
You will need to decide what unity-editor version you want to use for these steps. If you have an existing project, you can check out your `ProjectSettings/ProjectVersion.txt` file found within your unity project to determine what editor version you use. For this demonstration I'm going to choose to use editor version `2021.3.13f1`. I'm also going to stick with the `ubuntu` version of the docker images. You can check out what `unityci/editor` images are available for `2021.3.13f1` using this url: https://hub.docker.com/r/unityci/editor/tags?page=1&name=ubuntu-2021.3.13f1-

Go ahead and pull down some images
```bash
docker pull unityci/editor:ubuntu-2021.3.13f1-base-1.0.1
```

## Request Activation File
```bash
docker run \
  --interactive \
  --tty \
  --rm \
  --volume "${PWD}/activate:/activate" \
  --workdir /activate \
  unityci/editor:ubuntu-2021.3.13f1-base-1.0.1 \
  bash -c 'unity-editor -logfile /dev/stdout -quit -createManualActivationFile'
```
This produces a directory called "activate" with a new file `Unity_v2021.3.13f1.alf` (2021.3.13f1 as this is the editor version I decided to use in the "Before Getting Started" section). Don't version control this file! You now need to turn this `.alf` file into a unity license. To do this, follow steps 2 and 3 described by game.ci: https://game.ci/docs/github/activation#converting-into-a-license

Go ahead and place the corresponding `Unity_v2021.x.ulf` file in the same `activate` directory as the `.alf` file.

## Creating a New Unity Project
```bash
cat << 'HERE' > /tmp/script.sh
unity-editor \
    -logFile /dev/stdout \
    -quit \
    -manualLicenseFile /UnityLicense.ulf
unity-editor \
    -logFile /dev/stdout \
    -quit \
    -createProject /unity/sampleProject
HERE

docker run \
  --interactive \
  --tty \
  --rm \
  --env UNITY_EMAIL \
  --env UNITY_PASSWORD \
  --volume "${PWD}/activate/Unity_v2021.x.ulf:/UnityLicense.ulf" \
  --volume "${PWD}/unity:/unity" \
  --volume "/tmp/script.sh:/script.sh" \
  --workdir /unity \
  unityci/editor:ubuntu-2021.3.13f1-base-1.0.1 \
  bash -c 'chmod +x /script.sh; /script.sh'
```
At the end of this process, you'll have a new directory `unity` with a new unity project called `sampleProject`. Nice! You can open this project locally in Unity Hub.

You may want to take this opportunity to add a unity specific `.gitignore` for your newly created unity project.

## Running Tests
```bash
cat << 'HERE' > /tmp/script.sh
unity-editor \
    -logFile /dev/stdout \
    -quit \
    -manualLicenseFile /UnityLicense.ulf

unity-editor \
    -logFile /dev/stdout \
    -quit \
    -projectPath /unity/sampleProject \
    -runTests \
    -testResults /test_results/edit_mode_results.xml \
    -testPlatform EditMode \
    -enableCodeCoverage \
    -debugCodeOptimization

unity-editor \
    -logFile /dev/stdout \
    -quit \
    -projectPath /unity/sampleProject \
    -runTests \
    -testResults /test_results/play_mode_results.xml \
    -testPlatform PlayMode \
    -enableCodeCoverage \
    -debugCodeOptimization
HERE

docker run \
  --interactive \
  --tty \
  --rm \
  --env UNITY_EMAIL \
  --env UNITY_PASSWORD \
  --volume "${PWD}/activate/Unity_v2021.x.ulf:/UnityLicense.ulf" \
  --volume "${PWD}/unity:/unity" \
  --volume "/tmp/script.sh:/script.sh" \
  --volume "${PWD}/builds:/builds" \
  --workdir /unity \
  unityci/editor:ubuntu-2021.3.13f1--1.0.1 \
  bash -c 'chmod +x /script.sh; /script.sh'
```
## Producing Builds
