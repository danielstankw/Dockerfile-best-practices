# Dockerfile Best Practices 🐳

Writing production-ready Dockerfiles is not as simple as you could think about it. 

This repository contains some best-practices for writing Dockerfiles. Even though there is plentora of articles describing best practices, some of them are outdated or lack lesser known details - this repo aims to breach that gap. 
This is all guidance, not a mandate - there may sometimes be reasons to not do what is described here, but if you _don't know_ then this is probably what you should be doing.

### Disclaimer ⚠️
___
This is a compilation of best practices learned during my short career, read online and in books. If you find mistakes or would like to add/ clarify something feel free to create pull request.  
Throughout this file you will find the following ❌ **Bad:** and ✅ **Good:**. Using the approach listed under *Bad* isn't necessarily a mistake, but it's less optimal than *Good*.

## List of Content 📋

The following are included in the Dockerfile in this repository:

1. [Use official Docker images whenever possible](#1-use-official-docker-images-whenever-possible)
2. [Limit Image Layers](#2-limit-image-layers)
3. [Do NOT use `latest` tag, choose specific image tag](#3-do-not-use-latest-tag-choose-specific-image-tag)
4. [Only Store Arguments in `CMD` (cmd vs entrypoint)](#4-only-store-arguments-in-cmd-cmd-vs-entrypoint)
5. [Use `COPY` instead of `ADD`](#5-use-copy-instead-of-add)
6. [Combine `apt-get update` and `apt-get install`](#6-combine-apt-get-update-and-apt-get-install)
7. [Run as a Non-Root User](#7-run-as-a-non-root-user)
8. [Do not use a UID below 10,000](#8-do-not-use-a-uid-below-10000)
9. [Use static UID and GID](#9-use-static-uid-and-gid)
10. [Use multi-staged builds](#10-use-multi-staged-builds-to-reduce-final-image-size)
11. [Use `--no-cache-dir` (🐍-specific)](#11-use---no-cache-dir--specific)
12. [Order layers by change frequency](#12-order-layers-by-change-frequency---put-the-most-stable-commands-first)
13. [Use `.dockerignore`](#13-use-dockerignore)
14. [Set `WORKDIR` explicitly](#14-set-workdir-explicitly)
15. [Use Build time arguments for flexibility](#15-use-build-time-arguments-for-flexibility)
16. [Use `--chmod` in `COPY` instead of seperate `RUN`](#16-use---chmod-in-copy-instead-of-seperate-run)
17. [Use `--no-install-recommends` (🐍-specific)](#17-use---no-install-recommends--specific)
18. [Common performance optimizations](#18-common-performance-optimizations)
19. [Add metadata labels for better image management](#19-add-metadata-labels-for-better-image-management)
20. [Avoid `COPY .` whenever possible](#20-avoid-copy--whenever-possible)

___
## 1. Use official Docker images whenever possible

Official Docker images are reliable, secure, and optimized for size and performance. Maintained by experienced contributors, they follow best practices and come with community support. Explore [Python Official Images](https://hub.docker.com/_/python/tags).

## 2. Limit Image Layers
Minimize the number of layers to keep images lightweight and faster to build. Each `RUN` instruction in your Dockerfile will end up creating an additional cache layer in your final image. The best practice is to limit the amount of layers to keep the image lightweight.

✅ **Good:**
```dockerfile
RUN apt-get update && apt-get install -y \
    curl wget \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```
❌ **Bad:**
```dockerfile
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN apt-get clean
```
> **Tip**: Use `&&` to chain commands and a single `RUN` block for efficiency.  
> **Tip2**: Order the layers from one that is less likely to change, to one that will change more often.

## 3. Do NOT use `latest` tag, choose specific image tag
Using latest can lead to unpredictable behavior when the base image updates. If you don’t specify a specific version or tag in your Dockerfile, it will default to using the latest version of the image.

> **Note :  Specifying a version ensures consistency but requires manual updates to benefit from the latest security patches.**

✅ **Good:**
```dockerfile
FROM node:18.13.0
```
❌ **Bad:**
```dockerfile
FROM node:latest
```

## 4. Only Store Arguments in `CMD` (cmd vs entrypoint)
Use `CMD` for the default behavior or runtime arguments. Avoid hardcoding in `CMD`.
`CMD` should contain command arguments, while `ENTRYPOINT` should contain the command itself.

- `ENTRYPOINT`: Defines the main command to be executed when the container starts.
- `CMD`: Provides default arguments for the `ENTRYPOINT` or acts as the default command when `ENTRYPOINT` is not defined. It allows users to override arguments at runtime.

✅ **Good:**
```dockerfile
ENTRYPOINT ["python", "main.py"]
CMD ["--host=0.0.0.0", "--port=5000"]
```
❌ **Bad:**
```dockerfile
CMD ["python", "main.py", "--host=0.0.0.0", "--port=5000"]
```

**Use case:**  
To run with default arguments: 
```bashrc
docker run myapp
```
This will run with the defaults: `--host=0.0.0.0` and `--port=5000`.  
If we wish to change the port:  
```bashrc
docker run myapp --host=0.0.0.0 --port=8000
```


## 5. Use `COPY` instead of `ADD`
`COPY` is more explicit. Use `ADD` only when you need to automatically extract `tar` files or download a file from remote URLs.

✅ **Good:**
```dockerfile
COPY ./app /app/
```
✅ **Good:**
```dockerfile
# Extracting tar file and adding to the image
ADD app.tar.gz /app/
```
❌ **Bad:**
```dockerfile
ADD ./app /app/  # Use COPY instead
```

## 6. Combine `apt-get update` and `apt-get install`

__Prerequistite__ - *Package Index Files*  
Package index files are metadata files maintained by a package management system (such as `apt` in Debian-based systems) that contain information about the available software packages i.e: package names, versions, dependancies and sources.  
These files are essential for package updates. On devian based systems they are located in `/var/lib/apt/lists/`

__General Rule__
Always combine `apt-get update` with `apt-get install` to ensure you're installing the latest available packages.

The `apt-get update` command fetches the latest package lists. These lists contain information about the available packages and their versions and are stored in that cache layer.
If you run `apt-get install` in a new `RUN` command the package index from previous layer is no longer accessible during installation, meaning it will use an outdated package index leading to:
- Installing outdated packages
- Dependancy failures

>**Remember** Every `RUN` line in Dockerfile is a different process.

✅ **Good:**
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends\
    package1 \
    package2 \
    && rm -rf /var/lib/apt/lists/*
```
❌ **Bad:**
```dockerfile
RUN apt-get update
RUN apt-get install -y --no-install-recommends package1 package2 && rm -rf /var/lib/apt/lists/*
```
>**Tip** To reduce the image size, remove the packge index files after installation using: `rm -rf /var/lib/apt/lists/*`

## 7. Run as a Non-Root User
Running containers with a non-root user is a critical security best practice that helps prevent container breakout attacks and limits potential damage from compromised applications.

> **Note: When setting up your container's directory structure, it's important to establish proper ownership and permissions before switching to a non-root user.**

__Key points:__
- Create a dedicated user and group with specific IDs
- Set up directory structure and permissions before switching users
- Use `--chown` flag with `COPY` command to maintain correct ownership
- Apply minimal required permissions 
- Switch to non-root user

✅ **Good:**
```dockerfile
FROM python:3.12-slim

# Create app user and group with specific IDs for consistency
RUN groupadd -g 10001 appgroup && \
    useradd -u 10000 -g appgroup appuser

WORKDIR /app

# Set up directory structure with proper permissions first
RUN mkdir -p /app/logs /app/data /app/config && \
    chown -R appuser:appgroup /app && \
    chmod -R 755 /app && \
    chmod -R 775 /app/logs  # Writable for logs

# Install dependencies as root
COPY --chown=appuser:appgroup requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code with correct ownership
COPY --chown=appuser:appgroup . .

# Switch to non-root user
USER appuser:appgroup

CMD ["python", "app.py"]
```

❌ **Bad:**
```dockerfile
FROM python:3.12-slim

# Missing user creation and running as root
WORKDIR /app

# Incorrect permissions handling
COPY . .
RUN chmod 777 -R /app  # Too permissive!

# No user specification - defaults to root
CMD ["python", "app.py"]
```



## 8. Do not use a UID below 10,000
UIDs below 10,000 are a security risk on several systems, because if someone does manage to escalate privileges outside the Docker container their Docker container UID may overlap with a more privileged system user's UID granting them additional permissions. For best security, always run your processes as a UID above 10,000.

✅ **Good:**
```dockerfile
RUN groupadd -g 10001 appuser && \
    useradd -u 10001 -g appuser appuser
```
❌ **Bad:**
```dockerfile
RUN groupadd -g 100 appuser && \
    useradd -u 100 -g appuser appuser
```

## 9. Use static UID and GID
- Files and directories on a Linux system are associated with specific UIDs and GIDs, which determine who can read, write, or execute them (`rwx`).  
- When a Docker container creates or manipulates files on a shared volume or directly on the host filesystem, the files are owned by the UID/GID of the container process that created them.
- If container uses dynamically assigned UIDs/GIDs (the default), the container’s UID/GID could vary between builds or deployments.
- This variation in UIDs/GIDs makes it harder to manage file ownership consistently, especially when these files need to be accessed or modified on the host system.

✅ **Good:**
```dockerfile
ARG UID=10001
ARG GID=10001

RUN groupadd -g $GID appuser && \
    useradd -u $UID -g appuser appuser
```
❌ **Bad:**
```dockerfile
RUN adduser --system appuser  # Random UID/GID assigned
```

## 10. Use multi-staged builds to reduce final image size
Multi-stage builds are a powerful technique to create smaller, more secure Docker images by separating build-time dependencies from runtime requirements.  
Assume we have:
```
├── app.py              # Main application code
├── requirements.txt    # Dependencies
└── .gitignore
```

>**NOTE: `requirements.txt` should have dependencies versions pinned ex. `requests==2.31.0`**

✅ **Good: Single-Stage Build**
Simple and straightforward. Includes all build tools and depandencies in the final image, resulting in larger final image size. 
```dockerfile
FROM python:3.12-slim
WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
CMD ["python", "app.py"]
```

🔥✅ **Better: Multi-Stage Build**
Uses two stages, in first stage `builder` installs dependancies in virtual env, while in the second `runner` stage copies only the necessarly files.  

**Why use virtual environment?**  
In multi-staged builds, we need to copy dependancies from `builder` to the final `runner` stage. By default when installing Python packages they and related files are installed in various places. By using virtual env we know exactly where those dependancies are located and therefore copying them over from one stage to another is a simpler task. [Read more](https://pythonspeed.com/articles/multi-stage-docker-python/)

```dockerfile
FROM python:3.12-slim as builder

WORKDIR /app
# Create virtual env in /opt/venv which isolated Python packages from system Python
RUN python3 -m venv /opt/venv
# Modifies PATH and puts the venv bin directory as first in PATH
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.12-slim as runner

# Copy venv installed packages from builder to runner stage
COPY --from=builder /opt/venv /opt/venv
# Adds venv to PATH in runner stage
ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /app
COPY . .
CMD ["python", "app.py"]
```

**[Explanation](https://pythonspeed.com/articles/activate-virtualenv-dockerfile/)**  
The most important part is setting PATH: PATH is a list of directories which are searched for commands to run. `activate` simply adds the virtualenv’s `bin/` directory to the start of the list, so when python command is executed system first checks the `/opt/venv/bin`, where it find our venv python and uses it instead of system Python.  
**We can replace activate by setting the appropriate environment variables: Docker’s ENV command applies both subsequent RUNs as well as to the CMD.**


## 11. Use `--no-cache-dir` (🐍-specific)
When pip installs packages, it keeps a cache of downloaded wheel files and source distributions. This cache is unnecessary in Docker images since we don't need to reinstall packages. Removing the cache reduces the final image size significantly. Containers should be immutable - once built, they shouldn't change, therefore package cache is only useful for future installations, which won't happen in an immutable container

✅ **Good:**
```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
# Creates minimal layer with just the installed packages
```
❌ **Bad:**
```dockerfile
RUN pip install -r requirements.txt
# Creates larger layer with cache files (~100-200MB extra)
```

## 12. Order layers by change frequency - put the most stable commands first
Docker uses a layer caching system during builds. Organizing layers by change frequency dramatically improves build performance.

✅ **Good:**
1. Base Image: Rarely changes; placed first.
2. System Dependencies: Stable; cached after the first build.
3. Python Dependencies: Relatively stable but may change with requirements.txt; cached when requirements.txt is unchanged.
4. Application Code: Changes most frequently; placed last to minimize cache invalidation.

```dockerfile
# Use a base Python image
FROM python:3.11-slim
# Set a working directory
WORKDIR /app
# Install system dependencies (stable)
RUN apt-get update && apt-get install -y build-essential
# Install Python dependencies (semi-stable)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
# Copy application code (frequent changes)
COPY . .
# Set the entry point
CMD ["python", "app.py"]

```

❌ **Bad:**
```dockerfile
# Use a base Python image
FROM python:3.11-slim
# Copy application code (frequent changes)
COPY . .
# Set a working directory
WORKDIR /app
# Install Python dependencies (semi-stable)
RUN pip install --no-cache-dir -r requirements.txt
# Install system dependencies (stable)
RUN apt-get update && apt-get install -y build-essential
# Set the entry point
CMD ["python", "app.py"]
```

## 13. Use `.dockerignore`
The `.dockerignore` file prevents unnecessary files from being included in the build context, improving build performance and security.

```.dockerignore
# Version control
.git
.gitignore

# Development artifacts
__pycache__
*.pyc
*.pyo
*.pyd
.Python
env/
venv/
.env*
*.log

# Development tools
.idea/
.vscode/
*.swp
*.swo
```

## 14. Set `WORKDIR` explicitly

```dockerfile
WORKDIR /app             # Good
RUN cd /app && command   # Bad
```

## 15. Use Build time arguments for flexibility
Build-time arguments (ARG) provide a way to pass configuration options to your Docker build process. This is especially useful when you need different configurations for different environments (e.g., development, staging, production) without modifying the Dockerfile itself.

```dockerfile
ARG PORT=3000
EXPOSE ${PORT}
```

You can override it during the build process by running:
```
docker build --build-arg PORT=8080 -t myapp .
```


## 16. Use `--chmod` in `COPY` instead of seperate `RUN`
The `--chmod` flag in `COPY` or `ADD` allows you to set file permissions during the copy process, eliminating the need for additional `RUN` commands. This reduces the number of layers in your image and improves build performance.

✅ **Good:**
```dockerfile
COPY --chmod=755 script.sh .
```
❌ **Bad:**
```dockerfile
COPY script.sh .
RUN chmod +x script.sh
```


## 17. Use `--no-install-recommends` (🐍-specific)
By default, the` apt-get install` command installs recommended and suggested packages, which can lead to unnecessary bloat in your Docker image. Using the `--no-install-recommends` flag ensures that only the essential packages are installed, keeping your image smaller and faster to build.

✅ **Good:**
```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends package && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
```

>**Tip: After installing packages, always clean up apt caches to avoid unnecessary image bloat:**


## 18. Common performance optimizations
Each language and runtime has specific configurations that can significantly enhance performance when running in a containerized environment. These optimizations can reduce memory usage, improve execution speed, and make better use of container resources.

```dockerfile
# Node.js optimizations
ENV NODE_OPTIONS="--max-old-space-size=2048" \
    UV_THREADPOOL_SIZE=64 \
    NODE_NO_WARNINGS=1

# Python optimizations
ENV PYTHONUNBUFFERED=1 \        # Ensure logs and print() are written immediately to the console without being buffered.
    PYTHONDONTWRITEBYTECODE=1   # Disable generation of .pyc files when importing modules - save space and avoid unnecessary I/O


# Java optimizations
ENV JAVA_OPTS="-XX:+UseG1GC -XX:+UseContainerSupport -XX:MaxRAMPercentage=75"

# Golang optimizations
ENV GOGC=off \
    GOMAXPROCS=2
```

## 19. Add metadata labels for better image management
Labels provide descriptive metadata for your Docker images. They simplify image management, help with automation, and ensure compliance with standards. Labels are particularly useful for identifying the purpose, version, and maintainer of an image.

```dockerfile
LABEL maintainer="Daniel Jones <danielJones@gmail.com>" \
      description="Docker image for X application" \
      version="1.0" 
```

## 20. Avoid `COPY .` whenever possible
Using `COPY .` indiscriminately copies everything from the build context into the image, including unnecessary files like .git directories, local configuration files, or temporary files - unless those are excluded in `.gitignore`. 
Explicitly specify the files and directories you need.


# Sources 🔗
https://hynek.me/about/  
https://pythonspeed.com/articles/dockerizing-python-is-hard/  
https://hynek.me/articles/docker-uv/  
https://docs.docker.com/build/building/best-practices/  
https://pythonspeed.com/articles/base-image-python-docker-images/  
https://github.com/dnaprawa/dockerfile-best-practices  
https://sysdig.com/learn-cloud-native/dockerfile-best-practices/ 
