# Configure Container Registry

We will use the Docker Hub registry to push an image and a signature with cosign.

## Install and start Docker

To install docker on your Ubuntu system, follow the [Docker documentation](https://docs.docker.com/engine/install/ubuntu/#install-from-a-package).
Verify if the Docker daemon is running. If not,
```
sudo systemctl start docker
```

Let's create an image. You can use the following Dockerfile or any existing image you already have locally
```
cat > Dockerfile <<EOF
FROM alpine
CMD ["echo", "Hello Sigstore!"]
EOF
```
```
docker build -t sigstore-thw:latest .
```

Login to Docker Hub
```
sudo docker login
```

## Tag and push an image

Now we can tag and push our image:

```bash
docker tag SOURCE_IMAGE_NAME:VERSION ghcr.io/TARGET_OWNER/TARGET_IMAGE_NAME:VERSION
```

Push re-tagged imaged to the container registry:

```bash
docker push OWNER/IMAGE_NAME:VERSION
```

For example:
```
sudo docker push mayacostantini/sigstore-thw:latest

The push refers to repository [docker.io/mayacostantini/sigstore-thw]
994393dc58e7: Mounted from library/alpine 
latest: digest: sha256:67350a93198db86763c8db751138ed9392b3c44bd7afdb17f7130b5b73bd51c8 size: 528
```
