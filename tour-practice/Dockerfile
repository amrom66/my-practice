FROM golang:1.16-alpine

RUN export GOPROXY=https://goproxy.io,direct \
    &&go get golang.org/x/tour

EXPOSE 3999

CMD ["/go/bin/tour", "-http", "0.0.0.0:3999"]

