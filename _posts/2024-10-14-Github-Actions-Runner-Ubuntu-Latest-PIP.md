---
layout: post
title: GitHub Actions Runners - `ubuntu-latest` and PIP
date: 2024-09-13 20:00 -0400
description: >
  Earlier this morning, my team encoutered an error related to a breaking change in python while using the `ubuntu-latest` runner in GitHub Actions.  This post is a quick write-up on how we resolved the issue and what the root cause was.
tags: [GitHub, Actions, Ubuntu, Runners, PIP, Python]
---

Earlier this morning, my team encoutered an error related to a breaking change in python while using the `ubuntu-latest` runner in GitHub Actions.  This post is a quick write-up on how we resolved the issue and what the root cause was.

## tl;dr

[GitHub is currently rolling out an update to the `ubuntu-latest` runner label](https://github.blog/changelog/2024-09-25-actions-new-images-and-ubuntu-latest-changes/), slowly switching it from aliasing the `ubunutu-22.04` runner image to the `ubuntu-24.04` runner image.  This change has some breaking changes in it, which impacted my team this morning when trying to run an actions workflow that uses PIP to install dependencies.  We temporarily changed the `runs-on` declaration in the workflow to target `ubuntu-22.04` which offered us a temporary fix, but later we updated the workflow to install Python and dependencies ourselves.

This was an especially confusing issue because the `ubuntu-latest` runner image is supposed to be a stable image that is updated automatically by GitHub, so we were not expecting any breaking changes.  GitHub also did not explicitly state that this update would be breaking, so it was a bit of a surprise.  Oh, and since this is an update that's slowly rolling out (between September 25 and October 30) the issue may not be immediately apparent to everyone (which is what happened to us).  

_Our recommendation is that you always include setup steps in your GitHub Actions workflows to install your own dependencies._  This is a best practice, since it gives you the most control over your workflow and prevents you from being impacted by changes like this one.  Also if you had to run this workflow on a self-hosted runner, you would need to install Python and dependencies anyway, so it's a good practice to get into.

## The Issue

Python installations which are not fully managed by the user are called ["Externally Managed Environments"](https://packaging.python.org/en/latest/specifications/externally-managed-environments/).  A lot of linux distributions (including MacOS) ship with Python as part of the operating system (or installed by a package manager such as apt or homebrew), and these are considered externally managed environments.  Linux has a growing dependency on this Python distribution shipping as part of the OS, so there's a large uptick in seeing these Externally Managed Environments shipping natively.  Unfortunately, this also means that using a utility like PIP to install modules can cause issues in these environments, and can actually lead to breaking the entire underlying operating system.  For example, on a Mac if Python is installed via Homebrew, it's usually stored in a directory like `/opt/homebrew/opt/python@3.12/Frameworks/Python.framework/Versions/3.12/lib/python3.12` which is not a directory that PIP can natively write to (at least not outside of `sudo`) and also is pretty close to where some critical system files might be stored.  If you installed something directly from PIP that makes modifications to the Python installation, you could break the entire operating system.

Externally Managed Environments define a marker file named `EXTERNALLY-MANAGED` in the python installation directory.  This file is used to indicate that the Python installation is managed by the system and should not be modified by PIP.  If PIP detects this file, it will refuse to install packages into the Python installation directory and will instead display an error message (which is also stored in the marker file).

### So why does this impact GitHub Runners?

Ubuntu 24.04 ships with Python 3.12.3 and PIP 24.  The Ubuntu 22.04 image ships with Python 3.10.12 and PIP 22.  PIP started performing the Externally Managed Environment check starting in [version 23](https://discuss.python.org/t/announcement-pip-23-0-release/23342).  So in addition to the Python version change, the PIP version change is also causing issues.  The `ubuntu-latest` runner image is currently being updated to use the Ubuntu 24.04 image, which is why this issue is starting to surface now.

## Possible Solutions

Here's a few possible solutions to use to get around this problem.

1. **Use the `ubuntu-22.04` runner image directly** - This is the solution that we used.  It's a quick fix that will get you back up and running.  The main downside to this solution is - as stated before - the `ubuntu-22.04` image ships with an older version of Python (3.10.12) so you should still update your workflows to handle this case in the future.  And again - the rollout of the re-aliasing of `ubuntu-latest` to `ubuntu-24.04` is still ongoing, so you may not be impacted yet (or you may be intermittently impacted).

2. **Use virtual environments** - As a developer who works on python scripts, I'm a big fan of using [virtual environments (venv's)](https://docs.python.org/3/library/venv.html).  Virtual environments have lots of advantages, especially as a developer, since they are isolated environments that can be easily created and destroyed and contain their own sets of python interpreters and packages.  PIP will recognize it is being called from within a venv and will install python packages to the virtual environment without having to be told to do so.  The downside of this within GitHub Actions is that you have to remember to activate the virtual environment before running any python scripts, and every time you have a new `run` step in your GitHub Actions workflow you'll need to re-activate the environment.  This can be a bit of a pain and adds additional overhead to your workflow.

3. **Use the `--break-system-packages` flag with PIP** - PIP has a CLI flag (or environment variable) for [`--break-system-packages`](https://pip.pypa.io/en/stable/cli/pip_install/#cmdoption-break-system-packages) which will tell it to ignore the EXTERNALLY-MANAGED marker file and install the package anyway.  This option should be used with extreme caution, as it overrides a safety feature that is in place to prevent you from breaking your system.  If you're using this flag, you should be very sure that you know what you're doing and that you're not going to break anything.

4. **(BEST OPTION) Install your own python interpreter** - A simple fix in GitHub Actions is to just install your own Python interpreter using the `setup-python` action.  This sounds very counter-intuitive since Python 3.12 is already installed on the runner, but by installing Python yourself you can control the version and it also won't use the operating system Python installation.  This adds additional steps, but is the most bulletproof solution for this issue with GitHub Actions.  This solution also gives the advantage of portability since you can specify the exact version of Python you want to use in your workflow and not have to worry about the underlying runner image changing or _where_ the action is being run.

## Conclusion

It's a best practice to not depend on dependencies being installed on GitHub Actions Runners.  Even though Python is listed as being installed, it's not guaranteed to be there in the future, and the version of Python that is installed is also not guaranteed to be the same.  This is why it's a best practice to install your own Python interpreter and dependencies in your GitHub Actions workflows.  This will give you the most control over your workflow and will prevent you from being impacted by changes like the one that is currently rolling out with the `ubuntu-latest` runner image.  Additionally, if this workflow needed to be run on a self-hosted runner, you would need to install Python and dependencies anyway, so it's a good practice to get into.