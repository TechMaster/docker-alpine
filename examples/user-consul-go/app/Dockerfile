# build stage
FROM golang:alpine AS build-env
ADD ./src /src
RUN cd /src && go build -o main

# final stage
FROM minhcuong/alpine-consul
MAINTAINER Trịnh Minh Cường <cuong@techmaster.vn>

# Add s6 service
ADD root /

# Copy result binary go app from /src folder of build container to /app folder
COPY --from=build-env /src/main /app/