# Tic Tac Toe Game

Learn GitHub Actions through a fun little game.

We'd like to run our workflow on a specific label name, suppose that it's peacock. We can run the workflow based on that label name with the following single line that packs a punch: if: contains(github.event.pull_request.labels.*.name, 'peacock'). Here's how it works:

if: is the conditional that will decide if the job will run
contains() is a function that allows us to determine if a value like say, a label named "peacock", is contained within a set of values like say, a label array ["bug", "stage", "peacock", "feature request"]
github.event.pull_request.labels is specifically accessing the set of labels that triggered the workflow to run. It does this by accessing the github object, and the pull_request event that triggered the workflow.
github.event.pull_request.labels.*.name uses object filters to filter out all information about the labels, like their color or description, and lets us focus on just the label names.
```
name: Stage the app

on: 
  pull_request:
    types: [labeled]

env:
  DOCKER_IMAGE_NAME: ulankford-azure-ttt
  IMAGE_REGISTRY_URL: docker.pkg.github.com
  #################################################
  ### USER PROVIDED VALUES ARE REQUIRED BELOW   ###
  #################################################
  #################################################
  ### REPLACE USERNAME WITH GH USERNAME         ###
  AZURE_WEBAPP_NAME: ulankford-ttt-app
  #################################################

jobs:
  build:
    runs-on: ubuntu-latest

    if: contains(github.event.pull_request.labels.*.name, 'stage')
    ```
