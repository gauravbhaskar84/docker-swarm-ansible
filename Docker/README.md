## Dockerfile using scratch to have minimum image size:

```ruby
FROM scratch

ADD main /
CMD ["/main"]
```
## Dockerfile.alpine using the alpine image:

```ruby
FROM alpine:latest
RUN mkdir /app
WORKDIR /app
COPY . .
CMD ["/app/main"]
```

## Build an image and run the container:

```sh
$ cd Docker
$ docker build -t testapp:ver1.0 -f Dockerfile.scratch .
$ docker tag testapp:ver1.0 gauravbhaskar84/testapp1.0
$ docker push gauravbhaskar84/testapp1.0

## Now we can run the below in the Vagrant machine:

$ docker service create --name my-web1 --publish 8080:8080 --replicas 2 gauravbhaskar84/testapp1.0
$ docker service ls
```

## Test the app (it will work for all IPs (.10 to .14) - we can pick anyone to test)

```ssh
http://192.168.77.10:8080/works
http://192.168.77.10:8080/metrics
```
