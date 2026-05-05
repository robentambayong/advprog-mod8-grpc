# Module 8: High Level Networking - gRPC Tutorial

### Roben Joseph B Tambayong - 2406453594

## Reflection Questions

**1. What are the key differences between unary, server streaming, and bi-directional streaming RPC (Remote Procedure Call) methods, and in what scenarios would each be most suitable?**

* **Unary:** This is the standard request-response model. I send one request and wait for one response. It fits simple, one-off actions like fetching a profile or submitting a payment request (`PaymentService`).
* **Server Streaming:** I send one request, but the server sends back a steady stream of data. This is great when the server needs to push large datasets or live updates without making the client wait for everything to load at once. Good examples are downloading big files, getting live stock prices, or pulling transaction logs (`TransactionService`).
* **Bi-directional Streaming:** Both the client and server send continuous data streams to each other at the same time over one connection. It works best for highly interactive, real-time apps like multiplayer games, live document editing, or chat features (`ChatService`).

**2. What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption?**

When building a gRPC service in Rust, I need to handle security at a few different levels:
* **Data Encryption:** gRPC data is unencrypted by default. I have to set up TLS (Transport Layer Security) so the data is encrypted while traveling across the network. This stops man-in-the-middle attacks from reading the binary payloads. Tonic actually has built-in features to help configure TLS.
* **Authentication:** I have to verify exactly who is making the request. A common way to do this is passing token-based auth like JWTs through gRPC metadata (which works like HTTP headers). In Tonic, I can use Interceptors to grab and validate these tokens before the request is processed.
* **Authorization:** Once I know who the user is, I need to check if they actually have permission to run that specific RPC method. This means adding role-based access control (RBAC) to my service logic so I can block unauthorized actions.

**3. What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications?**

Handling bidirectional streams in Rust brings up some tricky concurrency and state management issues. First, managing multiple active streams at the same time means I need thread-safe state sharing. For example, to broadcast a chat message to multiple users, I need to share a list of active sender channels across Tokio tasks. This usually requires `Arc<Mutex<T>>` or `RwLock`, which can cause deadlocks if I am not careful. Second, dealing with network disconnects is tough. The server has to catch when a client drops off so it can clean up resources and prevent memory leaks. Also, there is the issue of backpressure. If a client sends messages way faster than the server can handle, it could end up overwhelming the whole system.

**4. What are the advantages and disadvantages of using `tokio_stream::wrappers::ReceiverStream` for streaming responses in Rust gRPC services?**

* **Advantages:** It works as a great bridge between Tokio's `mpsc` channels and the `Stream` trait that Tonic requires. It lets me spawn background tasks to do heavy lifting like querying databases and then drop the results into a channel. `ReceiverStream` then automatically pushes those results to the client as a gRPC stream. It makes writing concurrent streaming code a lot easier to read.
* **Disadvantages:** It does add a little bit of overhead because of the channel synchronization. Also, if I do not size the channel buffer correctly, things can go wrong. If the buffer is too big and clients read slowly, it wastes memory. If it is too small, it fills up and blocks the producer task.

**5. In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity, promoting maintainability and extensibility over time?**

To keep the codebase easy to maintain, I would break the architecture into clear layers.
1. **Proto Module:** I would keep the generated protobuf code in its own module or a completely separate crate. That way, multiple microservices can share the exact same API contracts.
2. **Service Layer (Transport):** The Tonic `impl` blocks (like `MyPaymentService`) should only deal with gRPC operations like taking in requests and formatting the final responses.
3. **Core Logic Layer:** The actual business logic belongs in separate struct implementations. The Tonic service would just call those core functions instead of doing the work itself.
4. **Dependency Injection:** Instead of hardcoding everything, I would pass shared resources like database connection pools or configs into the service structs when initializing them.

**6. In the `MyPaymentService` implementation, what additional steps might be necessary to handle more complex payment processing logic?**

Right now, my `MyPaymentService` just prints a message and forces a `success: true` response. To make this work in the real world, I would need to add a few things:
* **Input Validation:** Check that the `amount` is actually greater than zero and the `user_id` is formatted correctly.
* **External Integration:** Hook it up to an actual payment gateway API (like Stripe or Midtrans) using asynchronous HTTP calls to process the money.
* **Database Operations:** Securely save the transaction record into a database. I would use ACID transactions to make sure no money is lost if the service happens to crash midway.
* **Robust Error Handling:** Instead of just panicking or returning true every time, I need to handle real failures like network timeouts or insufficient funds. Then I would return the right gRPC `Status` codes, such as `Status::invalid_argument` or `Status::internal`.

**7. What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms?**

Using gRPC really pushes a "contract-first" design approach. Because all communication has to be strictly defined in `.proto` files, microservices share a clear and unchanging source of truth. This makes things much more interoperable. The Protobuf compiler can auto-generate client and server code for almost any language, like Rust, Java, Go, or Python. My team could build a super fast Rust backend, and another team could build a Node.js frontend, and they would talk to each other without a problem. It forces strong typing and backward compatibility, which removes a lot of the guessing games that usually happen with undocumented REST APIs.

**8. What are the advantages and disadvantages of using HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs?**

* **Advantages:** HTTP/2 handles multiplexing. This means multiple requests and responses can travel over a single TCP connection at the same time without blocking each other (solving the Head-of-Line blocking issue). It also uses HPACK header compression, which cuts down bandwidth waste a lot compared to HTTP/1.1. When comparing gRPC to WebSockets, HTTP/2 gives us structured and typed streaming right out of the box. WebSockets just give you a raw data pipe, forcing you to build your own message typing system from scratch.
* **Disadvantages:** HTTP/2 is binary instead of plain text. This makes it harder to quickly debug by hand using tools like Postman or basic `curl` commands (though tools like `grpcurl` can help). Also, web browsers do not have great native support for gRPC over HTTP/2 yet, so you usually have to run a proxy like gRPC-Web to connect web frontends to gRPC backends.

**9. How does the request-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness?**

REST is totally request-driven, meaning the client always has to ask the server for new info. To get real-time behavior in REST, a client has to rely on polling (asking the server for updates every few seconds). This is really inefficient, wastes bandwidth, and creates lag. On the flip side, gRPC's bidirectional streaming keeps a single connection open the whole time. Both the client and server can push messages instantly the moment an event happens. This keeps latency incredibly low and reduces the load on the server, making it way better for real-time apps.

**10. What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads?**

* **gRPC/Protobuf:** The schema-based approach guarantees strict typing and structure, so both sides know exactly what the data looks like. It packs down into a tight binary format, making it much faster to parse and send over the network. It also handles backward and forward compatibility easily using field tags (like `string user_id = 1`). The downside is that you have to run a compiler step to update your code every time the schema changes.
* **JSON/REST:** JSON is easy for humans to read, very flexible, and simple to change on the fly without compiling anything. But that flexibility means there is no strict contract. If the backend team changes a field name from `userId` to `user_id`, or swaps a number for a string, it can silently break the client app while it is running. JSON payloads also take up a lot more space and are slower to process than Protobuf binaries.