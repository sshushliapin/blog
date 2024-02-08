---
layout: post
title:  "SXA CLI Installation Troubleshooting"
date:   2024-02-08 16:08:00 +0200
tags: [sxa, sxa-cli]
---

While the installation process for SXA CLI appears straightforward and is documented in various blog posts (such as [here](https://www.markvanaalst.com/blog/sxa/installing-the-sitecore-sxa-cli) or [here](https://www.getfishtank.com/blog/install-sxa-theme-in-sitecore-docker-mvc-sxa-10)), I encountered several issues that necessitated additional steps. Below, I outline these challenges and their resolutions.

## Python Dependency

Surprisingly, neither the blog posts nor the [official Sitecore documentation](https://doc.sitecore.com/xp/en/developers/sxa/103/sitecore-experience-accelerator/add-a-theme-using-sxa-cli.html) mentioned Python as a prerequisite. Upon attempting to install SXA CLI using the `npm i -g @sitecore/sxa-cli` command, I encountered the following error:

```
npm ERR! gyp ERR! find Python ********************************************************
npm ERR! gyp ERR! find Python You need to install the latest version of Python.
npm ERR! gyp ERR! find Python Node-gyp should be able to find and use Python. If not,
npm ERR! gyp ERR! find Python you can try one of the following options:
...
```

Following the provided instructions, I downloaded and installed the latest Python version (3.12.2 as of February 6, 2024), ensuring to select 'Add python.exe to PATH' during installation.

![alt text](/assets/img/2024-02-08-python-versions.png)

Although this resolved the error, the Python-related issue persisted. More on this later. Let's address the next challenge.

## "Desktop development with C++" Workload

Another hurdle emerged due to the absence of the "Desktop development with C++" workload, a requirement that can be fulfilled through the Visual Studio Installer. 

```
npm ERR! gyp ERR! find VS ************************************************************
npm ERR! gyp ERR! find VS You need to install the latest version of Visual Studio
npm ERR! gyp ERR! find VS including the "Desktop development with C++" workload.
npm ERR! gyp ERR! find VS For more information consult the documentation at:
npm ERR! gyp ERR! find VS https://github.com/nodejs/node-gyp#on-windows
npm ERR! gyp ERR! find VS ************************************************************
```

Despite enabling this workload initially, the error persisted. Upon closer inspection of the logs, I discovered that the Windows SDK was missing. Selecting the appropriate checkbox in the installer resolved the issue.

![alt text](/assets/img/2024-02-08-vs-installer.png)

## Revisiting the Python Issue

With the Visual Studio obstacle cleared, a new Python-related error surfaced:

```
npm ERR! ModuleNotFoundError: No module named 'distutils'
npm ERR! gyp ERR! configure error
npm ERR! gyp ERR! stack Error: `gyp` failed with exit code: 1
```

Further investigation revealed that the `distutils` library had been deprecated and removed in Python 3.12. A comprehensive discussion on potential solutions can be found [here](https://stackoverflow.com/questions/69919970/no-module-named-distutils-but-distutils-installed). Ultimately, downgrading to Python 3.11.8 resolved the issue.