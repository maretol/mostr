.PHONY: build run clean

build:
	go build -o tmp/relay ./cmd/relay

run: build
	./tmp/relay

clean:
	rm -rf tmp
