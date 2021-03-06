---
title: Preview a Jekyll site with Docker on Mac OS X
author: Carolyn Van Slyck <carolyn.vanslyck@rackspace.com>
date: 2015-09-23
permalink: docs/tutorials/preview-jekyll-with-docker-on-mac/
description: Learn how to preview a Jekyll site in a Docker container, so that you do not need to install Ruby or Jekyll on your local machine
docker-versions:
  - 1.8.2
topics:
  - docker
  - intermediate
---

**Note:** This tutorial is for Mac OS X users. If you are using another operating system, follow
[the tutorial for Linux]({{ site.baseurl }}/docs/tutorials/preview-jekyll-with-docker-on-linux/) or
[the tutorial for Windows]({{ site.baseurl }}/docs/tutorials/preview-jekyll-with-docker-on-windows/) instead.

[Jekyll][jekyll] is a popular static site generator, most commonly used for blogging on GitHub Pages.
It is written in Ruby and sometimes getting your machine setup properly to preview your
changes before publishing can be difficult.

This tutorial describes how to preview a Jekyll site in a Docker container, so that
you do not need to install Ruby or Jekyll on your local machine.

[jekyll]: https://jekyllrb.com/

## <a name="prerequisites"></a> Prerequisites
* [Docker Toolbox][docker-toolbox]
* An existing Jekyll site on your local file system. If you do not have
  an existing site, a good way to get started quickly is to download a [Jekyll theme][jekyll-themes].

    You must place your site in a sub-directory of **/Users**,
    though it can be more deeply nested, for example, **/Users/myuser/repos/my-site**.
    The **Users** directory is the only directory exposed by default from your local machine
    to the Docker host via VirtualBox. If you want to use a different directory,
    you must manually share it with the Docker host by using VirtualBox

[docker-toolbox]: https://www.docker.com/toolbox
[jekyll-themes]: https://github.com/jekyll/jekyll/wiki/Themes

## <a name="steps"></a> Steps

1. Open a command terminal and change to your Jekyll site directory.

2. Create a new file named **Dockerfile** and populate it with the following content.

    **Note**: If you are using GitHub Pages or are not using any Jekyll plug-ins,
    you do not need to create a custom Docker image. Skip to the next step.

    ```
    FROM grahamc/jekyll:latest

    # Install whatever is in your Gemfile
    WORKDIR /tmp
    ADD Gemfile /tmp/
    ADD Gemfile.lock /tmp/
    RUN bundle install

    # Change back to the Jekyll site directory
    WORKDIR /src
    ```

3. Create a script named **preview**. You might want to customize the
    `DOCKER_MACHINE_NAME` and `DOCKER_IMAGE_NAME` variables defined at the top
    of the file.

    ```bash
    #!/usr/bin/env bash

    # Set to the name of the Docker machine you want to use
    DOCKER_MACHINE_NAME=default

    # Set to the name of the Docker image you want to use
    DOCKER_IMAGE_NAME=my-site

    # Stop on first error
    set -e

    # Create a Docker host
    if !(docker-machine ls | grep "^$DOCKER_MACHINE_NAME "); then
      docker-machine create --driver virtualbox $DOCKER_MACHINE_NAME
    fi

    # Start the host
    if (docker-machine ls | grep "^$DOCKER_MACHINE_NAME .* Stopped"); then
      docker-machine start $DOCKER_MACHINE_NAME
    fi

    # Load your Docker host's environment variables
    eval $(docker-machine env $DOCKER_MACHINE_NAME)

    if [ -e Dockerfile ]; then
      # Build a custom Docker image that has custom Jekyll plug-ins installed
      docker build --tag $DOCKER_IMAGE_NAME --file Dockerfile .

      # Remove dangling images from previous runs
      docker rmi -f $(docker images --filter "dangling=true" -q) > /dev/null 2>&1 || true
    else
      # Use an existing Jekyll Docker image
      DOCKER_IMAGE_NAME=grahamc/jekyll
    fi

    echo "***********************************************************"
    echo "  Your site will be available at http://$(docker-machine ip $DOCKER_MACHINE_NAME):4000"
    echo "***********************************************************"

    # Start Jekyll and watch for changes
    docker run --rm \
      --volume=$(pwd):/src \
      --publish 4000:4000 \
      $DOCKER_IMAGE_NAME \
      serve --watch --drafts -H 0.0.0.0
    ```

4. Mark the preview script as executable.

    ```bash
    $ chmod +x preview
    ```
5. Execute the preview script to start Jekyll.

    ```bash
    $ ./preview
    ```

6. In a web browser, navigate to the URL specified in the output.
    Following is an example of the output:

    ```bash
    ***********************************************************
      Your site will be available at http://192.168.99.100:4000
    ***********************************************************
    Configuration file: /src/_config.yml
                Source: /src
           Destination: /src/_site
          Generating...
                        done.
     Auto-regeneration: enabled for / src
    Configuration file: /src/_config.yml
        Server address: http://0.0.0.0:4000/
      Server running... press ctrl-c to stop.
    ```
<br/>

You can now modify your site and preview the changes.
After you save a file, it will be automatically regenerated by Jekyll.
Refresh the page in your web browser to see your changes.

[jekyll-image]: https://hub.docker.com/r/grahamc/jekyll/

## <a name="troubleshooting"></a>Troubleshooting
You might encounter the following issue when running the preview script.

### <a name="troubleshooting-missing-config"></a> The \_config.yml file is not picked up by Jekyll
If you see output as follows where the configuration file is none,
the most likely cause is that the [Docker data volume][docker-volume], which exposes the Jekyll site on your local file
system to the Docker container, is configured incorrectly.

```text
Configuration file: none
            Source: /src
       Destination: /src/_site
      Generating...
```

Verify that the Jekyll site is located in a directory that is exposed via VirtualBox shared folders, for example, **/Users**.
See the [Prerequisites](#prerequisites) section for additional information.

[docker-volume]: https://docs.docker.com/userguide/dockervolumes/

## <a name="resources"></a>Resources

* [Jekyll documentation](https://jekyllrb.com/docs/home/)
* [Using Jekyll with GitHub Pages](https://jekyllrb.com/docs/github-pages/)
