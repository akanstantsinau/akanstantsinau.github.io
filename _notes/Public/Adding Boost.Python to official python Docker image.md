---
title: Adding Boost.Python to official python Docker image
feed: show
date: 11-10-2022
permalink: /adding-boost.python-to-official-docker-image
format: list
---

#blog #docker #python
# Adding Boost.Python to official python Docker image

## Overview
The Docker community provides official docker images for various Python versions https://hub.docker.com/_/python
If your project uses libraries based on Boost.Python, you need appropriate libboost-python library compiled excactly for used Python version.

This article considers Debian-based Docker images.

## Issue
Assume you use Python 3.10 and need a Boost.Python library.
The idea may be to use `python:3.10-slim` base image and then install libboost-python via `apt`:
```
FROM python:3.10-slim

RUN apt-get update && \
    apt-get install -yq \
    libboost-python && \
    rm -rf /var/lib/apt/lists
```

This is bad approach, because:
- Base image is based on Debian Bullseye release
- Bullseye is based on python 3.9. So any python-related libraries installed will be linked to python 3.9
- In base image system `python` is replaced to python3.10, but `apt` packages database is obviously not changed.
You can check that after building docker image described above, your system will have `libboost_python39` library installed, and it will not work with python3.10

## Right way
You should download Boost sources and build `libboost-python`  that matches your python version(or use Python that is provided by distro -for Bullseye it is 3.9)
```
FROM python:3.10-slim

RUN apt-get update && \
    apt-get install -yq \
    build-essential wget && \
    rm -rf /var/lib/apt/lists

WORKDIR /opt
RUN wget -q https://boostorg.jfrog.io/artifactory/main/release/1.76.0/source/boost_1_76_0.tar.gz
RUN tar -xf boost_1_76_0.tar.gz && rm boost_1_76_0.tar.gz

WORKDIR boost_1_76_0
RUN ./bootstrap.sh --libdir=/usr/lib/$(dpkg-architecture -qDEB_HOST_MULTIARCH)
RUN ./b2 -d0 --with-python install
```
Note that Bullseye comes with Boost 1.74.0, but we use 1.76.0 because of this issue that was fixed in 1.76.0:
https://github.com/boostorg/python/commit/cbd2d9f033c61d29d0a1df14951f4ec91e7d05cd

After building this image you will have `libboost-python310` and system is consistent.

## Conclusion
You can freely choose base Docker image with desired python version, but any python libs installed with `apt` may not be compatible. You should build Boost.Python for your python version from sources. 