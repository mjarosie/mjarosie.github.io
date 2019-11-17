---
categories: shogun docker-toolbox windows
date:   2019-10-16
layout: post
title:  "Building Shogun on Windows using Docker Toolbox"
---

![Shogun with Docker Toolbox](/assets/2019-10-06-building-shogun-with-docker-toolbox-on-windows/shogun-with-docker-toolbox.jpg)

Hacktoberfest came early this year and I've finally decided to try my best with contributing to an open-source project. I ended up trying to hack around Shogun library which is one of the oldest (the first commit was created in [June 2006](https://www.openhub.net/p/shogun)) and largest open source machine learning frameworks (next to such obvious giants as TensorFlow or PyTorch). It's written in C++ and available on all major platforms. It also serves an interface with many other languages, such as Python, C#, Java, R or Ruby (among others). It's accomplished by SWIG, an interface compiler that connects programs written in C/C++ with other languages.

Initially I decided to develop on Windows by building source code and running tests on Docker container (using the [shogun-dev Docker image](https://hub.docker.com/r/shogun/shogun-dev)). "Why?" - you ask. I wanted to use WSL (Windows Subsystem for Linux) which would be an opportunity to get familiar with it. As it later turned out WSL still has problems with connecting with Docker machine running on Windows host, but I was already deep enough in the setup that I decided to carry on, learning a lot about Docker as I moved along.

As that took me a few evenings to make it fully work, I'm leaving a note here for myself in the future and anyone who would like to contribute to Shogun as well and would like to have a similar setup. It wouldn't have taken that much time if I'd have decided to just set up a Ubuntu virtual machine and work fully from there (which I ended up doing anyway). But well, then I wouldn't be able to say I've made it, right?

# Requirements

I assume that you've got these already:

- Windows version not supporting [Docker for Windows](https://docs.docker.com/docker-for-windows/install/#system-requirements) (if your system does support `Docker for Windows` - use it, there's no reason for using legacy solutions if you have a choice)
- [Docker Toolbox](https://docs.docker.com/toolbox/toolbox_install_windows/) installed
- You have checked out the repository on you host machine:

```
git clone && cd shogun && git submodule update --init
```

# TL;DR

**Don't**. Set up Ubuntu virtual machine and work from there. You'll save yourself hours of pain and tears.

# "I want to do it anyway!" version

## VirtualBox setup

- Increase available memory for docker-machine up to 4-5GB (if not it'll lead to `g++: fatal error: Killed signal terminated program cc1plus`)
- Add shared directory if you hold source code out of C:/Users (the only directory shared by default):
![VirtualBox directory sharing](/assets/2019-10-06-building-shogun-with-docker-toolbox-on-windows/vbox_dir_sharing.png)
- Enable creating symlinks from within the shared directory: `vboxmanage setextradata <name_of_virtual_machine> VBoxInternal2/SharedFoldersEnableSymlinksCreate/<name_of_shared_directory> 1` (in my case it was `default` and `c/source/` respectively)
- Confirm with: `VBoxManage getextradata <name_of_virtual_machine> enumerate`
- Run `Docker Quickstart Terminal` as admin (see <https://github.com/docker/toolbox/issues/619>)

## Shogun build

Pull the official shogun-dev Docker image (containing all packages necessary to successfully build the project):

```
docker pull shogun/shogun-dev
```

Inside the container we'll invoke `CMake` command in a specific directory (`-w` option), sharing the host directory (`/c/source/shogun` in my case) with the Docker container (`/shogun/` in my case). We'll use the `shogun/shogun-dev` image that we just pulled from the Docker repository. We're not going to be reusing the container, hence `--rm` option which causes it to be removed right after it stops. You'll need to specify CMake options that suit your needs. In my case I wanted to run tests (`-DENABLE_TESTING`) and build meta examples (`-DBUILD_META_EXAMPLES`). I also wanted to build Python interface (`-DINTERFACE_PYTHON` and `-DPYTHON_EXECUTABLE:FILEPATH`). After invoking `make install` command shogun library will be installed to `/shogun/build/opt` (`-DCMAKE_INSTALL_PREFIX` setting).

```
docker run -e CC=gcc -e CXX=g++ --rm --mount "type=bind,src=/c/source/shogun,target=/shogun" -w /shogun/build shogun/shogun-dev cmake -DENABLE_TESTING=ON -DCMAKE_INSTALL_PREFIX=/shogun/build/opt -DBUILD_META_EXAMPLES=ON -DINTERFACE_PYTHON=ON -DPYTHON_EXECUTABLE:FILEPATH=/usr/bin/python3 ..
docker run -e CC=gcc -e CXX=g++ --rm --mount "type=bind,src=/c/source/shogun,target=/shogun" -w /shogun/build shogun/shogun-dev make
docker run -e CC=gcc -e CXX=g++ --rm --mount "type=bind,src=/c/source/shogun,target=/shogun" -w /shogun/build shogun/shogun-dev make meta_examples
docker run -e CC=gcc -e CXX=g++ --rm --mount "type=bind,src=/c/source/shogun,target=/shogun" -w /shogun/build shogun/shogun-dev make install
```

Now we can start a shell session in Docker container to try out Python interface of the newly built library (note the necessary double slash):

```
docker run --rm -it -v //c/source/shogun/:/shogun shogun/shogun-dev bash
```

From there you'll need to set the `LD_LIBRARY_PATH` variable with wherever the `make install` command has produced the output, e.g.:

```
export LD_LIBRARY_PATH="/shogun/build/opt/lib/:$LD_LIBRARY_PATH"
```

Then point add the Python interface to the `PYTHONPATH`, in my case it was:
```
export PYTHONPATH="/shogun/build/src/interfaces/python/:$PYTHONPATH"
```

Then you'll need to get rid of dysfunctional symlinks:

```
rm -rf /shogun/build/examples/meta/data # That's a symlink which doesn't work when is within a volume which is shared between Windows host and Docker container…
cp -R /shogun/data/toy /shogun/build/examples/meta/data
```

And FINALLY you'll be able to run a meta example using Python:

```
cd /shogun/build/examples/meta/python/base_api/
python3 put_get_add.py
```

# What does work?

After going through the whole shebang described above, you'll be able to perform:

- compilation
- running meta examples and using the library (as far as I've checked)

# What doesn't work?

As I mentioned earlier, it's not a fully working setup and I don't think it's even possible. Because of symlinks involved you won't be able to run unit tests.

Good luck!