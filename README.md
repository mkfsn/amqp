# **pack.ag/amqp**

[![Go Report Card](https://goreportcard.com/badge/pack.ag/amqp)](https://goreportcard.com/report/pack.ag/amqp)
[![Coverage Status](https://coveralls.io/repos/github/vcabbage/amqp/badge.svg?branch=master)](https://coveralls.io/github/vcabbage/amqp?branch=master)
[![Build Status](https://travis-ci.org/vcabbage/amqp.svg?branch=master)](https://travis-ci.org/vcabbage/amqp)
[![GoDoc](https://godoc.org/pack.ag/amqp?status.svg)](http://godoc.org/pack.ag/amqp)
[![MIT licensed](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/vcabbage/amqp/master/LICENSE)

pack.ag/amqp is an AMQP 1.0 client implementation for Go.

[AMQP 1.0](http://docs.oasis-open.org/amqp/core/v1.0/os/amqp-core-overview-v1.0-os.html) is not compatible with AMQP 0-9-1 or 0-10, which are
the most common AMQP protocols in use today. A list of AMQP 1.0 brokers and other
AMQP 1.0 resources can be found at [github.com/xinchen10/awesome-amqp](https://github.com/xinchen10/awesome-amqp).

This library aims to be stable and worthy of production usage, but the API is still subject to change. To conform with SemVer, the major version will remain 0 until the API is deemed stable. During this period breaking changes will be indicated by bumping the minor version. Non-breaking changes will bump the patch version.

## Install

```
go get -u pack.ag/amqp
```

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md).

## Example Usage

``` go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"pack.ag/amqp"
)

func main() {
	// Create client
	client, err := amqp.Dial("amqps://my-namespace.servicebus.windows.net",
		amqp.ConnSASLPlain("access-key-name", "access-key"),
	)
	if err != nil {
		log.Fatal("Dialing AMQP server:", err)
	}
	defer client.Close()

	// Open a session
	session, err := client.NewSession()
	if err != nil {
		log.Fatal("Creating AMQP session:", err)
	}

	ctx := context.Background()

	// Send a message
	{
		// Create a sender
		sender, err := session.NewSender(
			amqp.LinkTargetAddress("/queue-name"),
		)
		if err != nil {
			log.Fatal("Creating sender link:", err)
		}

		ctx, cancel := context.WithTimeout(ctx, 5*time.Second)

		// Send message
		err = sender.Send(ctx, amqp.NewMessage([]byte("Hello!")))
		if err != nil {
			log.Fatal("Sending message:", err)
		}

		sender.Close(ctx)
		cancel()
	}

	// Continuously read messages
	{
		// Create a receiver
		receiver, err := session.NewReceiver(
			amqp.LinkSourceAddress("/queue-name"),
			amqp.LinkCredit(10),
		)
		if err != nil {
			log.Fatal("Creating receiver link:", err)
		}
		defer func() {
			ctx, cancel := context.WithTimeout(ctx, 1*time.Second)
			receiver.Close(ctx)
			cancel()
		}()

		for {
			// Receive next message
			msg, err := receiver.Receive(ctx)
			if err != nil {
				log.Fatal("Reading message from AMQP:", err)
			}

			// Accept message
			msg.Accept()

			fmt.Printf("Message received: %s\n", msg.GetData())
		}
	}
}
```

### Other Notes

By default, this package depends only on the standard library. Building with the
`pkgerrors` tag will cause errors to be created/wrapped by the github.com/pkg/errors
library. This can be useful for debugging and when used in a project using
github.com/pkg/errors.
