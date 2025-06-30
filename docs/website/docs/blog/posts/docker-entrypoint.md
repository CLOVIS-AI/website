---
date:
  created: 2025-06-30
slug: docker-entrypoint-cmd
tags:
  - Docker
---

# Docker misconception: should I use ENTRYPOINT or CMD?

Docker offers two ways to control what is executed when a container starts, `ENTRYPOINT` and `CMD`. Often, they are confused and misused.

<!-- more -->

When creating container images using a `Dockerfile` or `Containerfile`, we have the choice between multiple ways to run arbitrary code.
When containerizing applications, we want the application to start when the container starts.
Both `ENTRYPOINT` and `CMD` allow to do so.

## First, why not use RUN?

`RUN` allows specifying code that runs _while the image is being built_, before it is deployed.

Instead, `ENTRYPOINT` and `CMD` specify code that will run _once the container is starting on the end-user's machine_.

## Misconception: ENTRYPOINT cannot be overridden

In old versions of Docker, it wasn't possible to override the entrypoint, and people avoided using it for this reason.

However, this has been incorrect for a long time.

=== "Docker run"

    ```shell
    docker run -it --entrypoint /bin/apt ubuntu
    ```

=== "Docker Compose"

    ```yaml
    services:
      apt:
        image: ubuntu
        entrypoint: /bin/apt
    ```

=== "Kubernetes"

    ```yaml
    spec:
      containers:
        - name: apt
          image: ubuntu
          command: [ "/etc/apt" ]
    ```

    !!! danger
        Note that Kubernetes inverts the naming of the argument. Kubernetes' `command` means `ENTRYPOINT` in Docker speak, not `CMD`! For more information, see the [Kubernetes documentation](https://web.archive.org/web/20210514074330if_/https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/#notes).

=== "GitLab CI"

    ```yaml
    apt:
      image:
        name: ubuntu
        entrypoint: [ "/bin/apt" ]
    ```

Note that the syntax for overriding the `ENTRYPOINT` is different from overriding the syntax for overriding the `CMD`:


=== "Docker run"

    ```shell
    docker run -it ubuntu /bin/apt
    ```

=== "Docker Compose"

    ```yaml
    services:
      apt:
        image: ubuntu
        command: [ "/bin/apt" ]
    ```

=== "Kubernetes"

    ```yaml
    spec:
      containers:
        - name: apt
          image: ubuntu
          args: [ "/etc/apt" ]
    ```

=== "GitLab CI"

    ```yaml
    apt:
      image:
        name: ubuntu
        script:
          - /bin/apt
    ```

## What is the difference?

In the examples shown above, using `ENTRYPOINT` and `CMD` seem identical, and the behavior of each of them is almost identical.

In fact, `ENTRYPOINT` and `CMD` are not alternatives to each other, but are meant to be used together: the real command that Docker executes when the container starts is the concatenation of both.

`ENTRYPOINT`

:   The entrypoint of a container is the executable that represents the ‘essence' of the container.
    In a distribution image (like `ubuntu` or `alpine`) it is often set to `/bin/sh -c` or a similar shell, as the container exists to provide access to an environment.

:   In an application container, like `python` or `mongodb`, the entrypoint is set to the application itself, like `/bin/python` or `/bin/mongodb`.

The user will probably want to provide arguments to the container, which they will do by overriding the `CMD`. Since the `ENTRYPOINT` is provided separately, users do not need to specify the path to the application executable themselves if it is set as the `ENTRYPOINT`.

`CMD`

:   The command of a container is the default action that a container should do if the user has not specified anything.

:   The command is meant to be easy to override and is the dedicated location for users to place their custom arguments for the executable.

## Examples

If I was creating a container image for Python, I would probably write:
```Dockerfile
# …

ENTRYPOINT [ "/bin/python" ]
CMD [ ]
```
The `CMD` is empty because Python opens an interactive environment if no arguments are passed, which is the most useful behavior when nothing is specified.

As another example, here is an extract of [a Dockerfile I wrote](https://gitlab.com/opensavvy/system/simutrans-server/-/blob/main/src/Dockerfile?ref_type=heads) for a [Simutrans](https://www.simutrans.com/en/) server:
```Dockerfile
# …

ENTRYPOINT [ "/opt/simutrans/simutrans-server", "-server", "-set_pkdir", "/usr/share/simutrans/pak128", "-nomidi", "-nosound" ]
CMD [ "--help" ]
```

The `-set_pkdir /usr/share/simutrans/pak128` option is passed in the `ENTRYPOINT` because it allows the server executable to find its own files. If the user overrode this value, the server wouldn't be able to start.

Here, the `--help` command is specified to force users to select different arguments to actually launch the server.

## Conclusion

- Use `CMD` to specify the default command that an application should do.
- Use `ENTRYPOINT` to specify the executable itself, as well as any mandatory arguments. 
