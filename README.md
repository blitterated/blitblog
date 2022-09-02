# Building the blog in Docker with Hugo

[Repo for Docker file and whatnot](https://github.com/blitterated/hugodocker)

## Build and Run commands

### Build the Docker image

```sh
docker build -t hugoblog .
```

### Run the image with an interactive terminal

```sh
docker run -it --rm --init \
           -v "$(pwd)":/blog \
           -w /blog  --name hugo \
           -p 1313:1313 \
           hugoblog
```

### Start the server in the container

```sh
hugo server -D --bind "0.0.0.0"
```

[Link for site in container](http://localhost:1313)

#### Run the image with a one off command

```sh
docker run --rm \
           -v "$(pwd)":/blog \
           -w /blog  --name hugo \
           hugoblog "-c" "ls ."
```

