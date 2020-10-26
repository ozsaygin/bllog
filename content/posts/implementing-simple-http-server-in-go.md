---
title: "Implementing Simple Http Web Server in Go"
date: 2020-09-29T18:35:51+03:00
draft: true
tags: ["golang", "http"]
mermaid: true
---

I just have started to learning Go and was looking for a fun side-project
can play around with the standard library. I saw this
[post](https://theprogrammershangout.com/resources/projects/http-project-guide/intro.md)
in a discord server called [TPH](https://theprogrammershangout.com)
that explicitly defines the spesifications step-by-step to create a http
server. I thought that creating a http server might give more insights about both
how applications communicate with servers and contributes my learning journey in Go.
So, I decided to follow this guideline.

A http server is nothing but a server-side implementation of client-server architecture 
with socket API and quite similar to what we implemented in our network courses. 
In a server-client communication, server side process structure is composed of six stages. The server 
binds and starts listening the port. As new connections arrives server accepts those connections and 
handles them in separete threads. Each thread is basically a composition of data transfer operations
between server and client using `Send()` and `Receive()` methods.

{{<mermaid>}}
graph LR
Bind --> Listen
Listen --> Accept
Accept --> Listen
Accept --> Recv
Recv --> Send
Send --> Close
Send --> Recv
{{</mermaid>}}

Let's start with creating a baseline for our package.

In our **httpproto** package each server instance is derived from `Server`
struct encapsulating server address and port number.

```golang

type Server struct {
	Address string
	Port    int
}

```

Once a server object is created, it starts to serve with `Serve()` function. Basically, what we
design here is that implementing server part of a server-client application by using socket
communication API. Server starts to listen given network address with `net.Listen()`. Then,
we accept connection requests from HTTP clients with `Accept()` function. Each accepted connection
should be handled seperately because we do not want that an client access http message of another
client. Here, for each client we create a go routine with `go handleConnection(conn)` by passing
client's conn to the handler.

```golang
// Serve starts http serve which accept multiple connections
func (s *Server) Serve() {

	addr := s.Address + ":" + strconv.Itoa(s.Port)
	line, err := net.Listen("tcp", addr)

	if err != nil {
		log.Fatalf("Server could not listen %s...", addr)
	}

	// Start to accept incoming connections
	for {
		conn, err := line.Accept()
		if err != nil {
			log.Printf("Connection from %s could not connect to server...", addr)
		}
		// Handle the connection in a separate go routine
		go handleConnection(conn)
	}
}
```

Each go routine runs an instance of `handleConnection()` function inside with their own stack.

```golang
func handleConnection(conn net.Conn) {

    defer conn.Close()

	for connected := true; connected; {

		reader := bufio.NewReader(conn)
		buff := make([]byte, reader.Size())

		_, err := reader.Read(buff)

		if err == io.EOF {
			continue
		}

		if err != nil {
			log.Printf("Cannot read the buffer: %s", err)
		}

		data := string(buff)
		request := make(map[string]string)

		lines := strings.Split(data, "\n")
		if !(len(lines) > 0) {
			continue
		}

		for i, line := range lines {
			// request headers have "\r\n" chars at the end of line
			if i == 0 {
				header := strings.Split(line, " ")
				request["method"] = header[0]
				request["resource"] = header[1]
				request["version"] = strings.ReplaceAll(header[2], "\r", "")

			}

			if strings.Contains(line, ":") {
				line := strings.ReplaceAll(line, "\u0000", "")
				pair := strings.Split(strings.TrimSpace(line), ":")
				request[pair[0]] = pair[1]
			}
		}

		//fmt.Printf("Data received by server: \n%s", mapPrettier(request))
		// Process the request
		currentDir, err := os.Getwd()
		if err != nil {
			log.Println("Something bad happened while getting cwd")
		}

		resourceDir := "/www"
		resourcePath := strings.ReplaceAll(request["resource"], "../", "")
		path := currentDir + resourceDir + resourcePath

		switch request["method"] {
		case GetMethod:
			handleGetRequest(path, conn)

		case PostMethod:
			fmt.Println("POST method call")

		default:
			message := "HTTP/1.0 400 Bad Request\r\n\n"
			conn.Write([]byte(message))
			log.Println("HTTP/1.0 400 Bad Request")
        }
        conn.Close()
        connected = false
	}
}
```

### Testing

```golang

package core

import (
	"bufio"
	"fmt"
	"log"
	"net"
	"strconv"
	"testing"
)

func Test_handleGetRequest(t *testing.T) {
	type fields struct {
		address string
		port    int
	}

	tests := []struct {
		name   string
		fields fields
	}{
		{"local", fields{"127.0.0.1", 8080}},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
		})
	}
}

func TestServer_Serve(t *testing.T) {

	type fields struct {
		address string
		port    int
	}

	tests := []struct {
		name        string
		clientCount int
		fields      fields
	}{
		{"local", 1, fields{"127.0.0.1", 8080}},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {

			s := &Server{
				Address: tt.fields.address,
				Port:    tt.fields.port,
			}
			// Start server
			go s.Serve()
			addr := s.Address + ":" + strconv.Itoa(s.Port)

			conn, err := net.Dial("tcp", addr)
			if err != nil {
				log.Println("Cannot connect to server")
			}
			msg := `GET /a.txt HTTP/1.1\n
User-Agent: Mozilla/4.0 (compatible; MSIE5.01; Windows NT)\n
Host: www.tutorialspoint.com\n
Accept-Language: en-us\n
Accept-Encoding: gzip, deflate\n
Connection: Keep-Alive\n`

			conn.Write([]byte(msg))

			buf, err := bufio.NewReader(conn).ReadString('\n')
			if err != nil {
				fmt.Println("handle me")
			}
			fmt.Println(buf)

			fmt.Println("Client Body:" + buf)
			conn.Close()
			fmt.Println("client closed")
		})
	}
}

// TODO: Write test for each method call [GET, POST, PUT, ...]
// TODO: Write test for found, missing resources and corrupted header

```

You can find the whole project in [here](https://github.com/ozsaygin/httpproto).

<!-- ## Performance -->
