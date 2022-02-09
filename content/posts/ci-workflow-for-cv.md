+++ 
draft = false
date = 2022-02-09T00:53:33+08:00
title = "Building CI/CD workflow for my CV"
description = "Building a CI/CD workflow for my CV."
slug = ""
authors = ["Lyuwen Fu"]
tags = ["LaTeX", "CI/CD"]
categories = []
externalLink = ""
series = []
+++


In this project, I've built an CI/CD workflow with GitHub Actions to automatically build and deploy my CV to GitHub Pages whenever it's updated.


##### Docker Image of TeXLive

First of all, my CV is written in LaTeX the fundation of the workflow is building LaTeX files.
Thus an installation of TeXLive is needed as it's the standard TeX distribution.
However, a TeXLive distribution is massive and takes at least 10s of minutes to install,
and there is also the issue of the outdated TeX Live distribution that is available with Ubuntu's APT package manager.

A Docker image of an up-to-date TeXLive installation is an apparent solution to these problems. 
Although there is an existing `texlive/texlive` Docker image available on Docker Hub, a custom build is still desired for the optimal
image size and more control over what is installed within the image.

Therefore, the [fulvwen/texlive](https://hub.docker.com/r/fulvwen/texlive) is created with the CI workflow setup from GitHub Action to build
and push the image to Docker Hub automatically.
The repository [lyuwen/docker-texlive](https://github.com/lyuwen/docker-texlive) contains all the necessary information including
the `Dockerfile` and `texlive.profile` that defines the installed TeX packages, along with the GitHub Action configurations.
In the end, a Docker image of less than 1GB is created with most of the common components required to build LaTeX files.

Additionally, the script to install the TeXLive distribution can be find in the repository [lyuwen/install-tl-ubuntu](https://github.com/lyuwen/install-tl-ubuntu),
which is forked from [scottkosty/install-tl-ubuntu](https://github.com/scottkosty/install-tl-ubuntu) and modified so
it be used to install the latest distribution of TeXLive.


##### GitHub Action to build LaTeX files

For the purpose of making life easier, a Docker container GitHub Action is built to avoid redundant setup scripts to build.
With the Action [lyuwen/build-latex-action](https://github.com/lyuwen/build-latex-action), one can build a LaTeX source file into PDF
with minimum setup needed:

```yaml
uses: lyuwen/build-latex-action@v1
with:
  file: 'main.tex'
```

Initially, following the guide on [creating a Docker container action](https://docs.github.com/en/actions/creating-actions/creating-a-docker-container-action),
an intermediate Docker image to the accommodate entry point script is setup to be created on the fly when the GitHub Action is used.
However this build step takes nontrivial amount of time and should be avoided for efficiency, thus the
intermediate Docker image is now built and pushed to GitHub Package Registry [ghcr.io/lyuwen/build-latex-action:latest](https://github.com/lyuwen/build-latex-action/pkgs/container/build-latex-action/14664436?tag=latest).

##### CI/CD for my CV

With all necessary components prepared, now it's time to setup the CI/CD workflow for my CV.
The goal in this project is to build the CV from LaTeX source into a PDF file whenever the repository receives new pushes
and make the resulting PDF file available through GitHub Pages.

My CV uses a custom made document class derived from the regular `article` document class.
It is adapted from a XeLaTeX template [zachscrivena/simple-resume-cv](https://github.com/zachscrivena/simple-resume-cv),
with significant modifications for a much simpler and more compact design.

The workflow consists of the following parts:

* Check out the repository using standard `actions/checkout` action.
    ```yaml
    - uses: actions/checkout@v2
    ```
    
* Build LaTeX file with `lyuwen/build-latex-action` action.
    ```yaml
    - uses: lyuwen/build-latex-action@v1
      id: texbuild
      with:
        file: 'main.tex'
    ```
    
* Run a preparation script to copy the built PDF file and create a simple HTML index page into a directory for deployment.
    ```yaml
    - name: Prepare for publish
      run: |
        mkdir public
        cp ${{ steps.texbuild.outputs.output }} public/LFu_CV.pdf
        cd public && bash ../prepare_for_publish.sh
    ```

* Deploy the directory to GitHub Pages branch (`gh-pages`) of the repository using `peaceiris/actions-gh-pages` action.
    ```yaml
    - name: Deploy
      uses: peaceiris/actions-gh-pages@v3
      if: github.ref == 'refs/heads/main'
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_dir: ./public
    ```
    
* Finally, just to keep a record, the built PDF file is attached to the build as a build artifact.
    ```yaml
    - uses: actions/upload-artifact@v2
      with:
        name: ${{ steps.texbuild.outputs.output }}
        path: ${{ steps.texbuild.outputs.output }}
    ```
    
Now, as the final result, the CV can be found [here](https://www.lyuwenfu.me/lyuwen-cv/LFu_CV.pdf) with a simple index page
available [here](https://www.lyuwenfu.me/lyuwen-cv/) that shows the time in UTC when the latest CV is built.
The CV project repository is available in [lyuwen/lyuwen-cv](https://github.com/lyuwen/lyuwen-cv).
