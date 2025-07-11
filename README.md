## 🛰️ GoTraceroute

A flexible, cross-platform traceroute library and CLI tool written in Go.
It supports IPv4/IPv6 and TCP/UDP, and works on Linux, macOS, and Windows.
> Tested on Linux — please open an issue if you hit a bug on other platforms.


---

## ✏️ Features

✅ Supports IPv4 and IPv6.

✅ Supports TCP and UDP protcols.

✅ Works transparently across Unix and Windows (using golang.org/x/sys + build tags).

✅ Early stopping after N consecutive silent hops to speed up scans when destination is unreachable.

✅ Context-aware API for use as a library or CLI.

✅ Accurate results based on matching each response to sent probe request.


---

## ⚙️ Usage

### Dependency
- This tool depends on libpcap (on Linux/macOS) or Npcap SDK (on Windows) for capturing incoming TCP packets:

  	- On Linux:
	```bash
	sudo apt install libpcap-dev     // Ubuntu, Debian, Kali
 	sudo dnf install libpcap-devel   // Fedora
 	sudo yum install libpcap-devel   // CentOS, RHEL
 	sudo pacman -S libpcap           // Arch Linux, Manjaro
	```

  	- On macOS:
	```bash
	brew install libpcap
	```

  	- On Windows: Download from here [Npcap SDK](https://npcap.com/#download).



### Commnad-Line Interface:
#### Build and run
- Install it with `go install`:
```bash
go install github.com/0ne-zero/goTraceroute/cmd/gotraceroute@latest
```
- Build from source code:
```bash
git clone https://github.com/0ne-zero/goTraceroute.git
cd traceroute/cmd/
go build -o gotraceroute ./gotraceroute.go
sudo ./gotraceroute -h
```
> On Linux/macOS, raw sockets require sudo.
> On Windows, run from an elevated terminal.

- Download binary release: Download the appropriate binary for your OS and architecture from the [releases](https://github.com/0ne-zero/goTraceroute/releases) page.

### As a library (module):
- Get the module:
```bash
go get github.com/0ne-zero/goTraceroute
```

- Code example:
```go
package main

import (
	"fmt"
	"log"

	"github.com/0ne-zero/goTraceroute/pkg/core/hop"
	"github.com/0ne-zero/goTraceroute/pkg/core/options"
	"github.com/0ne-zero/goTraceroute/pkg/core/traceroute"
)

func main() {
	// Create traceroute options with defaults (max hops, timeouts, etc.)
	opts := options.NewTracerouteOptions()

	// Customize options as needed, (they've default value)
	opts.SetProbeProtocol(options.PROTOCOL_TCP) // Use TCP instead of default UDP
	opts.SetFirstHop(1)                         // Start from TTL=1
	opts.SetMaxHops(30)                         // Limit max TTL to 30 hops
	opts.SetTimeoutMs(5000)                     // 2 seconds timeout per probe
	opts.SetDelayMs(100)                        // 100 ms delay between probes
	opts.SetRetries(1)                          // Retry once if probe fails
	opts.SetMaxConsecutiveNoReplies(5)          // Stop early if 5 consecutive TTL probes get no replies (no ICMP or TCP response)

	// --- Synchronous traceroute ---

	// We pass a buffered channel sized to max hops, so traceroute can send results without blocking
	resChan := make(chan hop.TracerouteHop, opts.MaxHops())

	// Start traceroute and wait until it completes
	if err := traceroute.Traceroute("google.com", opts, resChan); err != nil {
		log.Fatal(err)
	}

	// Read results from the channel
	for h := range resChan {
		fmt.Printf("TTL %d\t%s\t%v\n", h.TTL, h.Address, h.ElapsedTime)
	}

	// --- Asynchronous traceroute ---

	// Useful if you want to show hops live or do other work concurrently
	resChan = make(chan hop.TracerouteHop)
	errChan := make(chan error)

	// Run traceroute in a separate goroutine, send any error to errChan
	go func() {
		if err := traceroute.Traceroute("google.com", opts, resChan); err != nil {
			errChan <- err
		}
	}()

	// Collect results as they arrive, or exit if an error occurs
	for {
		select {
		case hop, ok := <-resChan:
			if !ok {
				fmt.Println("Traceroute completed successfully")
				return // Channel closed → traceroute finished
			}
			fmt.Printf("TTL %d\t%s\t%v\n", hop.TTL, hop.Address, hop.ElapsedTime)
		case err := <-errChan:
			log.Fatal(err)
		}
	}
}
```

---

#### 📚 References

[RFC 792 — Internet Control Message Protocol (ICMP)](https://datatracker.ietf.org/doc/html/rfc792)
