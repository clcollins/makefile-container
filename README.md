# Using a Makefile to manage container-based projects

## INTRO (delete this title)
1. Why makefile, replace long commands with easy-to-remember commands
2. build, test, deploy, other helpers
3. Will stop if any step fails
4. check it out

## Targets

Make targets are (TARGET DEF).
Targets can be run using a simple command: `make <target-name>`.
For example,
Build a collection of targets for each function you might perform manually
Examples are building images, running tests, pushing to remote registries, etc.

## Build, Test, Deploy

1. Build the image - and tag approprately (git commit hash - could also automate a bit more and create tags)

```make
image_build:
  podman build --format docker -f Containerfile -t $(IMAGE_REF):$(HASH) .

```

What are these variables?

```make
# Image values
REGISTRY := "us.gcr.io"
PROJECT := "my-project-name"
IMAGE := "some-image-name"
IMAGE_REF := $(REGISTRY)/$(PROJECT)/$(IMAGE)

# Git commit hash
HASH := $(shell git rev-parse --short HEAD)
```

Ends up generating an image reference like `us.gcr.io/my-project-name/my-image-name:abc1234`.

Then tag the image as latest, since it's the latest. I don't generally use :latest myself, but it's easy enough to tag, and I use it in more public projects.

```make
image_tag:
  podman tag $(IMAGE_REF):$(HASH) $(IMAGE_REF):latest
```

2. Test, bin/smoke.py (spins up container, checks http reply (eg: 200), kills container)

So now the image has been built, and needs to be validated to make sure it meets some minimum requirements.
For my personal website, this is honestly just "does the webserver start and return something".
This could be accomplished with shell commands in the makefile, but it was easier for me to write a python script that started a container with [Podman](https://podman.io/), issues an http request to the container, verifies it receives a reply, and then cleans up the container.
Python's "try, except, finally" exception handling is perfect for this, and considerably easier than replicating the same logic from shell commands, in a Makefile.

```python3
#!/usr/bin/env python3

import time
import argparse
from subprocess import check_call, CalledProcessError
from urllib.request import urlopen, Request


parser = argparse.ArgumentParser()
parser.add_argument('-i', '--image', action='store', required=True, help='image name')
args = parser.parse_args()

print(args.image)

try:
    check_call("podman rm smk".split())
except CalledProcessError as err:
    pass

check_call(
    "podman run --rm --name=smk -p 8080:8080 -d {}".format(args.image).split()
)

time.sleep(5)

r = Request("http://localhost:8080", headers={'Host': 'chris.collins.is'})
try:
    print(str(urlopen(r).read()))
finally:
    check_call("podman kill smk".split())
```

This could definitley make this a more thorough test.
For example, during the build process, the git revision hash could be built into the response, and the test could check that the response included the expected hash.
This would have the benefit of verifying that there is at least some of the expected content.

3. Deploy
  a. push images to GCP registry
  b. rollout: GCP `gcloud run deploy` command
  c. could add validation: check for some git hash again? Roll back if not found?

If all goes well with the tests, then the image is ready to be deployed.
I use Google's Cloud Run service to host my personal website, and like any of the major cloud services, there is an excellent CLI tool that can be used to interact with the service.
Since Cloud Run is a container service, deployment consists of pushing the images built locally to a remote container registry, and then kicking off a rollout of the service using the `gcloud` cli tool.

The push can be done using Podman or Skopeo (or Docker, if you're using it).
My push target pushes the `$(IMAGE_REF):$(HASH)` image, and also the `:latest` tag.

```make
push:
  podman push --remove-signatures $(IMAGE_REF):$(HASH)
  podman push --remove-signatures $(IMAGE_REF):latest
```

After the image has been pushed, the `gcloud run deploy` command is used to rollout the newest image to the project, making the new image live.
Once again, the Makefile comes in handy here, because I can specify the `--platform` and `--region` arguments as variables in the makefile, so I don't have to remember them each time.
Let's be honest: I write so infrequently for my personal blog, there is zero chance I would remember these variables if I had to type them from memory each time I deployed a new image.

```make
rollout:
  gcloud run deploy $(PROJECT) --image $(IMAGE_REF):$(HASH) --platform $(PLATFORM) --region $(REGION)
```

## More targets

There are other helpful make targets as well.
When writing new stuff or testing CSS or code changes, I like to be able to see what I'm working on locally, without having to deploy it to a remote server.
For this, my Makefile has a `run_local` command, which will spin up a container with the contents of my current commit, and opens my browser to the URL of the page hosted by the locally-running webserver.

```make
.PHONY: run_local
run_local:
  podman stop mansmk ; podman rm mansmk ; podman run --name=mansmk --rm -p $(HOST_ADDR):$(HOST_PORT):$(TARGET_PORT) -d $(IMAGE_REF):$(HASH) && $(BROWSER) $(HOST_URL):$(HOST_PORT)
```

I use a variable for the browser name, too, so I can test with several if I wanted to.
By default, it will open in Firefox when I run `make run_local`.
If I wanted to test the same thing in Google, I could run `make run_local BROWSER="google-chrome"`.

When working with containers and container images, cleanup of old containers and images is an annoying chore, especially if your are iterating frequently.
I include targets in my Makefile for handling these tasks, too.
When cleaning up a container, if the container doesn't exist, Podman or Docker will return with an exit code of 125.
Unfortunately, make expects each command to return 0 or it stop processing, so I use a wrapper script to handle that case.

```make
#!/usr/bin/env bash

ID="${@}"

podman stop ${ID} 2>/dev/null

if [[ $?  == 125 ]]
then
  # No such container
  exit 0
elif [[ $? == 0 ]]
then
  podman rm ${ID} 2>/dev/null
else
  exit $?
fi
```

Cleaning images requires a bit more logic, but it can all be done within the Makefile.
To do this easily, I add a label (via the Containerfile) to the image when it's being built.
This makes it easy to find all the images with these lablels.
The most recent of these images can be identified by looking for the `:latest` tag.
Finally, all of the images, except those pointing to the image tagged with `:latest` can be deleted.

```make
clean_images:
  $(eval LATEST_IMAGES := $(shell podman images --filter "label=my-project.purpose=app-image" --no-trunc | awk '/latest/ {print $$3}'))
  podman images --filter "label=my-project.purpose=app-image" --no-trunc --quiet | grep -v $(LATEST_IMAGES) | xargs --no-run-if-empty --max-lines=1 podman image rm
```

This is the point where the use a Makefile for managing container projects really comes together into something cool.
To this point, the Makefile includes commands for building and tagging images, testing, pushing images, rolling out a new version, cleaning up container, cleaning up images, and running a local version.



2. Clean
  a. remove old containers.  I use cleanup.sh, but could just shell commands in makefile (these were more complicated); can be used to cleanup after running local containers too.
  b. Remove images just uses shell commands: get images (eval in this target, not global), podman rm images

3. Putting it all together: `build`, `test`, `deploy`, `clean`

4. Take it a step further: `make all`

## Conclusion

1. Why makefile
2. build, test deploy
3. CALL TO ACTION