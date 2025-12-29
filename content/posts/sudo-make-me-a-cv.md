+++
date = '2025-12-29T16:16:17+01:00'
title = 'Automatically build your CV with GitLab pipelines'
+++

## Background

For more than a decade I've been using a CV[^aka] I wrote in LaTeX; a few years ago I found myself in the unfortunate situation of having to update my CV while on a laptop that didn't have TeX Live installed, while being on a potato-quality, slower-than-a-56k-modem, internet connection.

I keep my TeX files in a git repository, in GitLab, so fetching those was not a problem, but fetching many hundreds of MBs for the compiler was.

While I ultimately[^facepalm] managed it, I also decided I didn't want to deal with the possibility, however remote, that this would happen again.

Since I was already using GitLab I decided to use their CI/CD pipelines to make this somewhat stable, and as soon as I had the opportunity I set up the most basic, sketchy-but-working pipeline that could do it: it used the [texlive/texlive](https://hub.docker.com/r/texlive/texlive) Docker image, built the artifacts, and then committed the PDFs to the repository, adding a [`[skip ci]`](https://docs.gitlab.com/ci/pipelines/#skip-a-pipeline) tag to prevent the pipeline from running forever.

I never really liked it, as it messed with the commits, so today I finally got around to rewriting the pipeline to build the CV and publish it[^internally] using GitLab releases, also adding direct links to the PDF (so I don't have to download a zipfile and decompress it).

(You can [skip to the code](#the-code) if you just want to see what I did and don't care about me rambling :) )

## The How

### Before you start

You must have runners set up for the CI/CD pipeline. Follow the official "[Use CI/CD to build your application](https://docs.gitlab.com/topics/build_your_application/)" docs from GitLab and you should be ok. I set mine up with the Docker runner.

### Building the PDF

My repository has everything in the root folder. It's not too much stuff, so I don't really care, and i `.gitignore`-d all the TeX Live temporary outputs[^clutter] to prevent them from cluttering the folder.

I had the build part down: this is the original `.gitlab-ci.yml` file I was using:

```yaml
---
variables:
  # Feel free to choose the image that suits you best.
  # blang/latex:latest ... Former image used in this template. No longer maintained by author.
  # listx/texlive:2020 ... The default, referring to TexLive 2020. Current at least to 2021-02-02.

  # Additional alternatives with high Docker pull counts:
  # thomasweise/docker-texlive-full
  # thomasweise/texlive
  # adnrv/texlive
  LATEX_IMAGE: texlive/texlive:latest
  CI_USERNAME: CI User
  CI_EMAIL: ci.user@example.com

# https://forum.gitlab.com/t/is-it-possible-to-commit-artifacts-into-the-git/22218/6
# https://marcosschroh.github.io/posts/autobumping-with-gitlab/
# https://gitlab.com/jasonrwang/dissertation-tudelft-latex/-/blob/master/.gitlab-ci.yml
# Token variable added via API: https://docs.gitlab.com/ee/api/project_level_variables.html

build:
  image: $LATEX_IMAGE
  before_script:
    - 'which git || ( apt-get update -qy && apt-get install git -qqy )'
    - git config --global user.email "${CI_EMAIL}"
    - git config --global user.name "${CI_USERNAME}"
  script:
    - latexmk -pdf
    - git add -f *.pdf # Force add PDF since we .gitignored it
    - git commit -m "Compiled PDF from $CI_COMMIT_SHORT_SHA [skip ci]" || echo "No changes, nothing to commit!"
    - git remote rm origin && git remote add origin "https://cicduser:${CV_BOT_TOKEN}@$REDACTED$/$CI_PROJECT_PATH.git"
    - git push origin HEAD:$CI_COMMIT_REF_NAME # Pushes to the same branch as the trigger

  artifacts:
    paths:
      - "*.pdf"

  rules:
    - changes:
      - "*.tex"
```

All I had to do was strip everything except for the "build the PDF" part; I took the opportunity to also rename the stage from `build` to `make_pdf` and remove the comments I no longer needed.

```yaml
variables:
  CV_PDF: CV.pdf  # The PDF you get by building CV.tex. Adjust accordingly.

# Filter for file changes.
.file-patterns-changes: &file-patterns-changes
  - "*.tex"

# Build the PDFs.
make_pdf:
  stage: build
  image: texlive/texlive:latest
  script:
    - latexmk -pdf
  artifacts:
    paths:
      - "${CV_PDF}"
  rules:
    - changes: *file-patterns-changes  # Only run if the correct files changed (must be added to all stages: https://gitlab.com/gitlab-org/gitlab/-/issues/370052)
```

I also split out the `rules.changes` section to be reusable (which turned out to be a great idea), by _borrowing_ from [this example](https://gitlab.com/Ambient.Impact/path-of-exile-2-item-filter/-/blob/main/.gitlab-ci.yml).

### Upload

Initially I thought I could just use a `release` stage and upload the files, but unfortunately GitLab's `release` stage will upload a zipfile of the whole "source code" (i.e. the repo), which means I won't just get my PDF, but will have to download everything, unzip the file, and get the PDF.

In reality, that's not a big deal, since the PDF is by far the heaviest file in the repo, but I know I can just attach files to a release (which is usually used to give people binaries for different OSes and let them download only the one they need) so let's figure out how to do that.

As it turns out, GitLab does have pretty good instructions on how to do that in their [Generic packages](https://docs.gitlab.com/user/packages/generic_packages/) docs[^butalso]: we add a stage for the upload, get the artifacts from the build job, and send them into the package registry.

> [!WARNING]
> I didn't have Package Registry enabled by default.
>
> If you don't either (which you can check by looking for `Deploy > Package registry` in your repo: if it's not there it's disabled) go to `Settings > General > Visibility, project features, permissions`, turn on "Package registry", and save.
>
> You do **not** need to enable `Allow anyone to pull from package registry`

With this set up, this is the relevant config for the upload stage (**this is not a complete configuration**, I'm just showing what I changed from the build example):

```yaml
variables:
  [...]
  # Use the CI_PIPELINE_IID as the version; it's autoincremented.
  PACKAGE_REGISTRY_URL: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/CV/v${CI_PIPELINE_IID}"

[...]

# Upload the PDFs to the Generic Package Registry/Repository.
# https://docs.gitlab.com/user/packages/generic_packages/
upload_pdf:
  stage: upload
  image: curlimages/curl:latest
  needs:                               # Run after the make_pdf job
    - job: make_pdf
      artifacts: true
  rules:
    - changes: *file-patterns-changes  # Only run if the correct files changed (must be added to all stages: https://gitlab.com/gitlab-org/gitlab/-/issues/370052)
  when: on_success                     # This is the default, but we're adding it explicitly
  script:
    - |
      curl --header "JOB-TOKEN: ${CI_JOB_TOKEN}" --upload-file ${CV_PDF} "${PACKAGE_REGISTRY_URL}/${CV_PDF}"
```

`PACKAGE_REGISTRY_URL` is set to

```text
"${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/CV/v${CI_PIPELINE_IID}"
```

You can change a lot of things here, but I'd recommend leaving the first part (`${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/`) alone, and changing

* `CV` which, as far as I can tell, is just the name of the package in the registry
* `v${CI_PIPELINE_IID}`
  * the `v` is there on purpose, to make the version look like `v1`, `v2` etc.
  * `CI_PIPELINE_IID` is the auto-incremented variable that [represents](https://docs.gitlab.com/ci/variables/predefined_variables/) the pipeline ID. I don't really care about the version, so this is a way to be able to forget about it.

Two things to note:

1. the `needs` section ensures that this runs after the `make_pdf` stage. I don't know why, but without it GitLab was running the `release` stage before the `build` one, which - obviously - wasn't working
2. the `rules` section **must** be there and it **must** be identical to **all** other stages. Otherwise GitLab will complain that `upload_pdf` needs `make_pdf`, but `make_pdf` does not exist. Why? No idea, but people in the [linked issue](https://gitlab.com/gitlab-org/gitlab/-/issues/370052) said it fixed it for them (and it did for me too)

At this point you should try and upload the `.gitlab-ci.yml` file and see if it throws any errors. If everything is working you can move on to the deployment, otherwise I'd recommend fixing whatever issue you have before doing that.

If all goes well, in `Deploy > Package registry` you should have a shiny new `CV` package, containing your PDF.

### Release

You could stop here; you can just get your PDF from Package Registry, you don't need to create a release (and waste disk space), but we're doing this properly here (and who's to say you can't reuse this code to release something else later?).

If you want to create a release you can use a `release` stage with the dedicated `glab`[^glab] image, and attach the PDF using GitLab's [Release assets as Generic packages](https://docs.gitlab.com/user/project/releases/#release-assets-as-generic-packages) instructions.

> [!NOTE]
> For some reason **every example** on GitLab's website doesn't tell you `glab` needs to authenticate **before** trying to release. It will cause all sorts of inconvenience until you realize the CI/CD pipeline already gives you everything you need to authenticate (i.e. a token and the host it can be used on).

So what you do is add the last stage:

```yaml
# Create a release that contains the PDFs as independent assets.
publish_pdf:
  stage: release
  image: registry.gitlab.com/gitlab-org/cli:latest
  needs:                                            # Run after the upload_pdf job
    - job: upload_pdf
      artifacts: true
  rules:
    - changes: *file-patterns-changes               # Only run if the correct files changed (must be added to all stages: https://gitlab.com/gitlab-org/gitlab/-/issues/370052)
  when: on_success                                  # This is the default, but we're adding it explicitly
  script:                                           # Auth doesn't happen automatically, so we need to do this before trying to create a release
    - |
      glab auth login --hostname $CI_SERVER_HOST --job-token $CI_JOB_TOKEN
    - |
      glab release create "${CI_PIPELINE_IID}" --name "Release ${CI_PIPELINE_IID}" \
        --assets-links="[{\"name\":\"${CV_PDF}\",\"url\":\"${PACKAGE_REGISTRY_URL}/${CV_PDF}\"}]"
```

As before we have a `rules` section and a `needs` block that forces a dependency on the `upload_pdf` stage having completed.

### Why is my pipeline looping?

At this point, if you push your code you'll spot an issue: the CI/CD pipeline runs twice: one on your `main` (or `master`) branch, and then once on a numbered branch, suspiciously numbered as your pipeline.

You can stop that (if you want) by adding this at the top of your YAML config:

```yaml
# Stop the creation of multiple (pointless) pipelines.
# https://forum.gitlab.com/t/how-to-stop-release-jobs-from-creating-new-pipelines/75683/5
workflow:
  rules:
    - if: $CI_COMMIT_TAG
      when: never
    - when: always
```

I don't understand exactly why that happens, something to do with releases creating tags (?); I'm sure it has a perfectly valid use case for most people, but it doesn't for me, so I just disabled it.

Now lint and save the `.gitlab-ci.yml` file ([here's my complete one](#the-code)), make a change to the `.tex` file, push and watch your pipeline make you a shiny new CV!

## Issues I ran into

Lest you think this was easy, I created a separate (blank) repository to do this and get the pipeline running. I knew I was going to run into issues and didn't want to "poison" the nice git log in my main repository just to get this working.

In the space of the ~4 hours it took me to do this, that "dev" repository gained:

* 45 commits
* 93 releases
* 192 pipeline runs
* and a total of 368 jobs

The issues I ran into (at least the ones I remember) were:

* **Pipeline loops**: I pushed a new version, went to get lunch, came back less than an hour later to a couple hundred new jobs
* **Release authentication**: as I mentioned above, none of the `glab` instructions I could find[^exceptauth] mentioned you had to authenticate the runner to the instance first. Thankfully `docker run registry.gitlab.com/gitlab-org/cli:latest glab help` and `[...] auth login --help` showed me the way.
* **The pipeline running inverted**: at some point `release_pdf` kept running before `make_pdf`. I don't understand why or how, but that's why I added the `needs` constraint, which then caused new issues
* **The `needs` issue**: why do I need to specify the same `rules` in all stages? I don't get it, especially if the pipelines are supposed to run sequentially. If the first one has a condition and the second one has a "`needs` the first one" section it seems pretty obvious to me what should happen? Regardless, I had to add the same `rules` section to all stages, which is where using `.file-patterns-changes` came in REALLY handy
* **`CI_COMMIT_TAG`**: I don't know how that comes about, but it was ALWAYS blank for me. I probably need to add it manually? I didn't want to, and not having it led to:
  * incorrect URLs for Package Registry, which manifested as `404` "the URL does not exist" errors that I couldn't understand until I changed `curl [...]` commands with `echo curl [...]` and saw `https://[...]/something//something` where the thing that was supposed to be between the `//` was the `CI_COMMIT_TAG`
  * pipelines failing at random and, despite the pipeline validator saying my pipeline was good...
  * ... random(-ish) failures as soon as I uploaded the new `.gitlab-ci.yml` file

## The Code

This is the complete code for my `.gitlab-ci.yml` file. As a reminder (or if you came here straight from the intro) my `.tex` file is in the root of the repository, and the compiled `.pdf` ends up having the same name as the `.tex` file, but with a different extension (in case it wasn't clear: `CV.tex` generates `CV.pdf`).

Adjust the relevant variables as needed.

```yaml
---
# Stop the creation of multiple (pointless) pipelines.
# https://forum.gitlab.com/t/how-to-stop-release-jobs-from-creating-new-pipelines/75683/5
workflow:
  rules:
    - if: $CI_COMMIT_TAG
      when: never
    - when: always

stages:
 - build
 - upload
 - release

variables:
  CV_PDF: CV.pdf  # The PDF you get by building CV.tex. Adjust accordingly.
  # Use the CI_PIPELINE_IID as the version; it's autoincremented.
  PACKAGE_REGISTRY_URL: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/CV/v${CI_PIPELINE_IID}"


# Filter for file changes.
.file-patterns-changes: &file-patterns-changes
  - "*.tex"

# Build the PDFs.
make_pdf:
  stage: build
  image: texlive/texlive:latest
  script:
    - latexmk -pdf
  artifacts:
    paths:
      - "${CV_PDF}"
  rules:
    - changes: *file-patterns-changes  # Only run if the correct files changed (must be added to all stages: https://gitlab.com/gitlab-org/gitlab/-/issues/370052)

# Upload the PDFs to the Generic Package Registry/Repository.
# https://docs.gitlab.com/user/packages/generic_packages/
upload_pdf:
  stage: upload
  image: curlimages/curl:latest
  needs:                               # Run after the make_pdf job
    - job: make_pdf
      artifacts: true
  rules:
    - changes: *file-patterns-changes  # Only run if the correct files changed (must be added to all stages: https://gitlab.com/gitlab-org/gitlab/-/issues/370052)
  when: on_success                     # This is the default, but we're adding it explicitly
  script:
    - |
      curl --header "JOB-TOKEN: ${CI_JOB_TOKEN}" --upload-file ${CV_PDF} "${PACKAGE_REGISTRY_URL}/${CV_PDF}"

# Create a release that contains the PDFs as independent assets.
publish_pdf:
  stage: release
  image: registry.gitlab.com/gitlab-org/cli:latest
  needs:                                            # Run after the upload_pdf job
    - job: upload_pdf
      artifacts: true
  rules:
    - changes: *file-patterns-changes               # Only run if the correct files changed (must be added to all stages: https://gitlab.com/gitlab-org/gitlab/-/issues/370052)
  when: on_success                                  # This is the default, but we're adding it explicitly
  script:                                           # Auth doesn't happen automatically, so we need to do this before trying to create a release
    - |
      glab auth login --hostname $CI_SERVER_HOST --job-token $CI_JOB_TOKEN
    - |
      glab release create "${CI_PIPELINE_IID}" --name "Release ${CI_PIPELINE_IID}" \
        --assets-links="[{\"name\":\"${CV_PDF}\",\"url\":\"${PACKAGE_REGISTRY_URL}/${CV_PDF}\"}]"
```

If you have multiple files/versions you want to upload you need to add multiple `curl` scripts in the `upload_pdf`, and multiple `--assets-links=[...]` in the `publish_pdf` section. I recommend also adding a separate variable for each version/file; you could do this with a `for` loop, but I'm not 100% sure how to build and attach the `--assets-links=` arguments to `glab release create`. I guess this exercise is left to the reader.

[^internally]: internally. It's not available online.
[^aka]: a.k.a. [_Curriculum Vitae_](https://en.wikipedia.org/wiki/Curriculum_vitae), resume, etc.
[^facepalm]: i.e. when I realized that "hey, you can SSH into your homelab and do it there, you idiot"
[^butalso]: And also in [this example](https://gitlab.com/gitlab-org/release-cli/-/tree/master/docs/examples/release-assets-as-generic-package), which is good but outdated (is uses `release-cli` for releasing instead of the newer [`glab`](https://docs.gitlab.com/cli/))
[^glab]: `registry.gitlab.com/gitlab-org/cli:latest`
[^exceptauth]: except for a "Caution" message in [this example](https://gitlab.com/gitlab-org/release-cli/-/tree/master/docs/examples/release-assets-as-generic-package), which was for the older CLI version so I didn't really pay too much attention to it. Also, the actual `release` YAML section doesn't need it, so I'm sure it'll be fine... Right?
[^clutter]: `$ cat .gitignore`

    ```text
    *.aux
    *.fdb_latexmk
    *.fls
    *.log
    *.out
    *.synctex.gz
    ````
