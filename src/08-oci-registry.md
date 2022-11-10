# Configure Container Registry

We will use the Quay.io container registry to push an image and a signature with cosign.

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

Login to Quay
```
sudo docker login quay.io
```

## Tag and push an image

Now we can tag and push our image:

```bash
docker tag SOURCE_IMAGE_NAME:VERSION TARGET_OWNER/TARGET_IMAGE_NAME:VERSION
```

Push re-tagged imaged to the container registry:

```bash
docker push OWNER/IMAGE_NAME:VERSION
```

For example:
```
sudo docker push mcostant/sigstore-thw:latest
```
