Dockerfile using scratch to have minimum image size:

```ruby
FROM scratch

ADD app /
CMD ["./app"]
````

Build an image and run the container:

```sh
$ cd Dockerized
$ docker build -t testapp:ver1.0 -f Dockerfile.scratch .
$ docker service create --name my-web1 --publish 8080:8080 --replicas 2 testapp:ver1.0
$ docker run -it -p 8080:8080 testapp:ver1.0  -- testing while running as a single container
```

Error thrown by the app:

``` ruby
standard_init_linux.go:211: exec user process caused "exec format error" 

```

Cause of the error:

```ruby
Looks like this app application was compiled on a different architecture, and because of that it is not working on Mac or Ubuntu type machines.
```