FROM golang:1.26.9-trixie

ENV DEBIAN_FRONTEND=noninteractive \
    CGO_ENABLED=1 \
    GOTOOLCHAIN=local

RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential ca-certificates curl git iproute2 jq openssl procps \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

RUN go mod download \
    && go build -o /usr/local/bin/nats-server . \
    && CGO_ENABLED=0 GOARCH=386 go build -o /dev/null .

# Compile every test binary so reproducers can run offline without rebuilding
# the dependency graph from scratch.
RUN go test -count=1 -run '^$' ./... \
    && go test -count=1 -race -run '^$' ./server

# Run a quick baseline (core protocol, config, auth) without failing the build.
# ./logger is omitted: its syslog tests need a syslog daemon the image lacks.
# The full suite (JetStream, clustering, MQTT, store) takes hours; see threat_model.md.
RUN go test -count=1 -p 1 -timeout 20m ./conf ./internal/... ./server/avl ./server/gsl ./server/stree \
    || echo 'WARNING: package baseline tests failed'
RUN go test -count=1 -timeout 20m \
      -tags=skip_store_tests,skip_js_tests,skip_mqtt_tests,skip_msgtrace_tests,skip_no_race_tests \
      -run '^(TestParse|TestConfig|TestOpts|TestAuth|TestNkey|TestJWT|TestSublist)' ./server \
    || echo 'WARNING: server baseline tests failed'

# New reproducers and patches must also build with the network disabled.
ENV GOPROXY=off \
    GOSUMDB=off

CMD ["bash"]
