+++
author = "Julien Vermillard"
title = "Life and death of Java asynchronous network programming"
tags = [ "Java", "network programming", "asynchronous"]
date = "2024-08-18"
+++
Network programming in Java has changed significantly over the last 20+ years. Mainly driven by the desire to handle larger loads by an ever-increasing number of connections.

In Java, the journey started with synchronous programming, using one thread per connection (to a server or a connected client). The principal limit was memory because each thread cost approximately half a megabyte (maybe less with some tuning).

This would look like:

```java
import java.io.*;
import java.net.*;

public class SimpleTcpServer {

    public static void main(String[] args) {
        int port = 1234;

        ServerSocket serverSocket = new ServerSocket(port);
        while (true) {
           // Accept a new client connection
           Socket clientSocket = serverSocket.accept();
           System.out.println("New connection from " + clientSocket.getInetAddress().getHostAddress());

           // Start a new thread to handle the connection
           new Thread(new ClientHandler(clientSocket)).start();
       }
    }
}

class ClientHandler implements Runnable {
    private final Socket clientSocket;

    public ClientHandler(Socket clientSocket) {
        this.clientSocket = clientSocket;
    }

    @Override
    public void run() {
        try {
            BufferedReader in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
            PrintWriter out = new PrintWriter(clientSocket.getOutputStream(), true);
            String message;
            while ((message = in.readLine()) != null) {
                System.out.println("Received: " + message);
                out.println("Echo: " + message);
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            clientSocket.close();
        }
    }
}
```
At the time (early 2000s), I was working on monitoring appliances running on 64 MB of RAM. The one thread per connection model worked fine until a new project arrived: I needed to connect to 1k IP devices to monitor them, and the classical one thread per connection model consumed twenty times more memory than I could afford.

## The C10K Problem

The C10K problem emerged in the late 1990s, highlighting the server’s challenge in handling more than 10,000 TCP connections. We started to see different proposals to change or evolve the traditional network programming model. The main idea was to remove the reliance on threads to reduce memory consumption and the massive performance impact of context switching.

See:
http://www.kegel.com/c10k.html
https://en.wikipedia.org/wiki/C10k_problem

## The Advent of NIO (Java 1.4)

In light of the problem raised by the C10K problem, Java 1.4 (released in 2002) introduced the New Input/Output (NIO) library, providing a non-blocking I/O model that significantly improved scalability. NIO allowed multiple connections (called Channels) to be managed by a single thread using selectors. Selectors would generate events when a connection is ready to be read (e.g., some data was received and buffered by the kernel), prepared to be written, or closed. The programming model changed from blocking synchronous threads to an event-driven one.

```java
// over simplified NIO event loop example:
Selectot selector = Selector.open();
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);
// Bind the socket to TCP server port
serverChannel.socket().bind("0.0.0.0:1234");
serverChannel.register(this.selector, SelectionKey.OP_ACCEPT);
System.out.println("Server bind and started…");
// event loop
while (true) {
   this.selector.select();
   Iterator keys = this.selector.selectedKeys().iterator();
   while (keys.hasNext()) {
      SelectionKey key = (SelectionKey) keys.next();
      // remove the processed event
      keys.remove();
      if (!key.isValid()) {
          continue;
      }
      if (key.isAcceptable()) {
          accept(key); // TODO
      } else if (key.isReadable()) {
          read(key); // TODO
      } else if (key.isWritable()) {
          write(key); // TODO
      }
   }
}
```
This new model allowed handling thousands of connections with a few threads, effectively addressing the C10K problem.

## The birth of Java asynchronous networking programming

While in C/C++, it’s frequently used to select/poll/queue directly. Java NIO is quite a complicated API and buggy (SSL engine quirks or spinning selector without events) to be used directly. If you skim Stackoverflow, you will see that it’s pretty challenging to get NIO right.

What complicates NIO is Sun’s attempt to unify Unix-style networking (BSD socket) with Microsoft Windows (Winsock). Also, event-driven programming in plain Java took a lot of work at the time. You were asking programmers used to synchronous POO to start to write asynchronous code, using mainly state machines (lambda/functional programming was not yet a thing in Java).

Example of an HTTP decoding state machine: https://github.com/netty/netty/blob/4.1/codec-http/src/main/java/io/netty/handler/codec/http/HttpObjectDecoder.java#L338

```java
        switch (currentState) {
        case SKIP_CONTROL_CHARS:
            // Fall-through
        case READ_INITIAL: try {
            ByteBuf line = lineParser.parse(buffer);
            if (line == null) {
                return;
            }
            final String[] initialLine = splitInitialLine(line);
            assert initialLine.length == 3 : "initialLine::length must be 3";

            message = createMessage(initialLine);
            currentState = State.READ_HEADER;
            // fall-through
        } catch (Exception e) {
            out.add(invalidMessage(message, buffer, e));
            return;
        }
        case READ_HEADER: try {
            State nextState = readHeaders(buffer);
            if (nextState == null) {
                return;
            }
            currentState = nextState;
            switch (nextState) {
            case SKIP_CONTROL_CHARS:
                // fast-path
                // No content is expected.
                addCurrentMessage(out);
                out.add(LastHttpContent.EMPTY_LAST_CONTENT);
                resetNow();
                return;
            case READ_CHUNK_SIZE:
                if (!chunkedSupported) {
                    throw new IllegalArgumentException("Chunked messages not supported");
                }
                // Chunked encoding - generate HttpMessage first.  HttpChunks will follow.
                addCurrentMessage(out);
                return;
            default:
                /*
                 * RFC 7230, 3.3.3 (https://tools.ietf.org/html/rfc7230#section-3.3.3) states that if a
                 * request does not have either a transfer-encoding or a content-length header then the message body
                 * length is 0. However, for a response the body length is the number of octets received prior to the
                 * server closing the connection. So we treat this as variable length chunked encoding.
                 */
                if (contentLength == 0 || contentLength == -1 && isDecodingRequest()) {
                    addCurrentMessage(out);
                    out.add(LastHttpContent.EMPTY_LAST_CONTENT);
                    resetNow();
                    return;
                }

                assert nextState == State.READ_FIXED_LENGTH_CONTENT ||
                        nextState == State.READ_VARIABLE_LENGTH_CONTENT;

                addCurrentMessage(out);

                if (nextState == State.READ_FIXED_LENGTH_CONTENT) {
                    // chunkSize will be decreased as the READ_FIXED_LENGTH_CONTENT state reads data chunk by chunk.
                    chunkSize = contentLength;
                }

                // We return here, this forces decode to be called again where we will decode the content
                return;
            }
        } catch (Exception e) {
            out.add(invalidMessage(message, buffer, e));
            return;
        }
        case READ_VARIABLE_LENGTH_CONTENT: {
            // Keep reading data as a chunk until the end of connection is reached.
            int toRead = Math.min(buffer.readableBytes(), maxChunkSize);
            if (toRead > 0) {
                ByteBuf content = buffer.readRetainedSlice(toRead);
                out.add(new DefaultHttpContent(content));
            }
            return;
        }
        case READ_FIXED_LENGTH_CONTENT: {
            int readLimit = buffer.readableBytes();

            // Check if the buffer is readable first as we use the readable byte count
            // to create the HttpChunk. This is needed as otherwise we may end up with
            // create an HttpChunk instance that contains an empty buffer and so is
            // handled like it is the last HttpChunk.
            //
            // See https://github.com/netty/netty/issues/433
            if (readLimit == 0) {
                return;
            }

            int toRead = Math.min(readLimit, maxChunkSize);
            if (toRead > chunkSize) {
                toRead = (int) chunkSize;
            }
            ByteBuf content = buffer.readRetainedSlice(toRead);
            chunkSize -= toRead;

            if (chunkSize == 0) {
                // Read all content.
                out.add(new DefaultLastHttpContent(content, trailersFactory));
                resetNow();
            } else {
                out.add(new DefaultHttpContent(content));
            }
            return;
        }
        /*
         * everything else after this point takes care of reading chunked content. basically, read chunk size,
         * read chunk, read and ignore the CRLF and repeat until 0
         */
        case READ_CHUNK_SIZE: try {
            ByteBuf line = lineParser.parse(buffer);
            if (line == null) {
                return;
            }
            int chunkSize = getChunkSize(line.array(), line.arrayOffset() + line.readerIndex(), line.readableBytes());
            this.chunkSize = chunkSize;
            if (chunkSize == 0) {
                currentState = State.READ_CHUNK_FOOTER;
                return;
            }
            currentState = State.READ_CHUNKED_CONTENT;
            // fall-through
        } catch (Exception e) {
            out.add(invalidChunk(buffer, e));
            return;
        }
        case READ_CHUNKED_CONTENT: {
            assert chunkSize <= Integer.MAX_VALUE;
            int toRead = Math.min((int) chunkSize, maxChunkSize);
            if (!allowPartialChunks && buffer.readableBytes() < toRead) {
                return;
            }
            toRead = Math.min(toRead, buffer.readableBytes());
            if (toRead == 0) {
                return;
            }
            HttpContent chunk = new DefaultHttpContent(buffer.readRetainedSlice(toRead));
            chunkSize -= toRead;

            out.add(chunk);

            if (chunkSize != 0) {
                return;
            }
            currentState = State.READ_CHUNK_DELIMITER;
            // fall-through
        }
        case READ_CHUNK_DELIMITER: {
            final int wIdx = buffer.writerIndex();
            int rIdx = buffer.readerIndex();
            while (wIdx > rIdx) {
                byte next = buffer.getByte(rIdx++);
                if (next == HttpConstants.LF) {
                    currentState = State.READ_CHUNK_SIZE;
                    break;
                }
            }
            buffer.readerIndex(rIdx);
            return;
        }
        case READ_CHUNK_FOOTER: try {
            LastHttpContent trailer = readTrailingHeaders(buffer);
            if (trailer == null) {
                return;
            }
            out.add(trailer);
            resetNow();
            return;
        } catch (Exception e) {
            out.add(invalidChunk(buffer, e));
            return;
        }
        case BAD_MESSAGE: {
            // Keep discarding until disconnection.
            buffer.skipBytes(buffer.readableBytes());
            break;
        }
        case UPGRADED: {
            int readableBytes = buffer.readableBytes();
            if (readableBytes > 0) {
                // Keep on consuming as otherwise we may trigger an DecoderException,
                // other handler will replace this codec with the upgraded protocol codec to
                // take the traffic over at some point then.
                // See https://github.com/netty/netty/issues/2173
                out.add(buffer.readBytes(readableBytes));
            }
            break;
        }
        default:
            break;
        }
```

This is more complex to write than a synchronous-pulling parser, which would look like:

```java
private static HttpRequest decodeHttpRequest(BufferedReader in) throws IOException {
    // Read the request line
    String requestLine = in.readLine();
    if (requestLine == null || requestLine.isEmpty()) {
        return null; // Invalid request
    }

    // Parse the request line
    String[] requestParts = requestLine.split(" ");
    if (requestParts.length != 3) {
        return null; // Invalid request line
    }
    String method = requestParts[0];
    String uri = requestParts[1];
    String httpVersion = requestParts[2];

    // Read and parse headers
    Map<String, String> headers = new HashMap<>();
    String headerLine;
    while ((headerLine = in.readLine()) != null && !headerLine.isEmpty()) {
        String[] headerParts = headerLine.split(":", 2);
        if (headerParts.length == 2) {
            headers.put(headerParts[0].trim(), headerParts[1].trim());
        }
    }

    // Return the decoded HTTP request
    return new HttpRequest(method, uri, httpVersion, headers);
}
```
## The advent of asynchronous network programming frameworks

Apache MINA, Netty, Grizzly, xNIO, and probably more frameworks were created. Several techniques were tried to make async network programming more accessible and reliable, from VertX to reactive programming frameworks. The reactive programming paradigm deals with async data streams and change propagation. It helps you set up a series of transformations and operations on data that will be executed as the data arrives and processed as it flows in.

RxJava TCP server example: https://github.com/whybe/rxnetty-tcp-sample/blob/master/src/main/java/io/reactivex/netty/samples/SimpleTcpServer.Java

Introducing this programming style in popular Java frameworks changed how we built enterprise applications: You can’t continue to use blocking-style JDBC drivers or HTTP clients. For example, introducing reactive programming in the Spring framework marked a significant evolution from the traditional blocking-style RestTemplate or JdbcTemplate.

This created a major API and code paradigm shift, enabling more significant parallelism and scalability. However, this came at the cost of increased complexity and an often misunderstood hard-to-debug threading model.

## Virtual thread. Asynchrony obsolete?

Java 21 (released in September 2023) introduced a new shift in network programming with the lightweight thread model. These new threads solve our initial connection scaling problem: memory cost! We go from a price range of hundreds of bytes to a few KBs.

Virtual threads are scheduled and preempted by the JVM in place of the O/S following the I/O event. For example, the scheduler will preempt a thread waiting for a network connection to receive some datagram, precisely like the O/S would do with a native thread, but at low memory cost while idling.

We can now use the 20-year-old synchronous programming model in a scalable way. This change alone makes the needs of async frameworks like Vert.x, Netty, Reactor, Flux… obsolete for most (or all?) use cases. Of course, after 20 years of errands in the realm of asynchronous programming and asynchronous framework, it’s a considerable disruption!

A myriad of libraries are not applicable anymore. You can now create a small Java 21 backend HTTP server from the std library: https://www.reddit.com/r/java/comments/18vysrr/jdk_http_server_handles_100000_reqsec_with_100_ms/

This causes quite some confusion in the Java community:

* Reddit: https://www.reddit.com/r/java/comments/1d43rbh/how_can_java_21_virtual_thread_replace_reactive/
* Netty GitHub issue: https://github.com/netty/netty/issues/12848
* Spring https://spring.io/blog/2022/10/11/embracing-virtual-threads#revision-of-programming-models

The confusion is in the air, but it makes me excited to return to Java after a long time, after looking at other ecosystems that embraced green/light threads, such as Go and Erlang/Elixir.

I’m happy we can use this opportunity to simplify our Java backend developer toolbox. We can remove a good chunk of the Spring framework layers and return to a simpler model: https://spring.io/blog/2023/07/13/new-in-spring-6-1-restclient

Async Network Programming served us well for 20 years as a way to mitigate the cost of threads in JVM, but it’s time to let it go and embrace a more accessible and manageable way.

