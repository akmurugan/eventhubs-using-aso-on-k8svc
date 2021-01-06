FROM golang as build-stage
WORKDIR /app

COPY go.mod go.sum /
RUN go mod download

COPY consumer.go /

RUN cd / && CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o consumer

FROM alpine
COPY --from=build-stage /consumer /
CMD ["/consumer"]