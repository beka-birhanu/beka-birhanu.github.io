---
title: "UDP Socket Manager"
date: 2026-05-17
---

This is a wrapper around Go's UDP package handling both the server and client sides of socket management. I built it back in 2024, right after I learned the basics of Go in my 3rd year of undergrad, while working on a simple game server that naturally required a UDP socket.

## Supported Features

- DTLS (symmetric encryption)
- Server session management
- Heartbeat-based session cleanup

## Record Structure and Receiving-Parsing

### Structure

```go
// Incoming bytes are parsed into the record struct
type record struct {
	Type byte
	Body []byte // up to max buffer size - 1
}
```

Types 1–5 are reserved for the manager itself: `ClientHelloRecordType` and `ServerHelloRecordType` for the handshake, `HelloVerifyRecordType` for address verification, and `PingRecordType`/`PongRecordType` for heartbeats. Anything else can be used for custom messages.

```go
const (
	ClientHelloRecordType byte = 1 << iota
	HelloVerifyRecordType
	ServerHelloRecordType
	PingRecordType
	PongRecordType
)
```

### Receiving-Parsing

When the server is started, it creates a channel to push raw `Record`s, which is consumed by `handleRawRecords`.

```go
// rawRecord is sent to the rawRecords channel when a new payload is received
type rawRecord struct {
	payload []byte
	addr    *net.UDPAddr
}

// parseRecord parses a byte slice into a record struct.
func parseRecord(r []byte) (*record, error) {
    ...
	return &record{
		Type: r[0],
		Body: r[1:],
	}, nil
}

// Serve starts listening to the UDP port for incoming bytes & then sends payload and sender address into the rawRecords channel if no error is found
func (s *ServerSocketManager) Serve() {
    ...
	s.rawRecords = make(chan rawRecord)
	go s.handleRawRecords()
    ...
}
```

The server will listen for incoming bytes and send them into the `rawRecords` channel, while trying to detect too-large payloads in a simple, probably stupid way.

```go
func (s *ServerSocketManager) Serve() {
	...
	for {
		select {
		case <-s.stop:
			return
		default:
			buf := make([]byte, s.readBufferSize+1) // Intentionally create more space than allowed for checking
			n, addr, err := s.conn.ReadFromUDP(buf)
			if n > s.readBufferSize { continue }
			s.rawRecords <- rawRecord{payload: buf[0:n], addr: addr}
		}
	}
}
```

The `handleRawRecords` method is responsible for consuming the `s.rawRecords` channel and parsing the incoming bytes into `Record`s. It looks at the first byte to know what type of record it is, then passes it to the responsible method.

```go
func (s *ServerSocketManager) handleRawRecords() {
	for r := range s.rawRecords {
		record, err := parseRecord(r.payload)
		switch record.Type {
		case ClientHelloRecordType:
			s.handleHandshakeRecord(r.payload, r.addr)
		case PingRecordType:
			s.handlePingRecord(r.payload, r.addr)
		default:
			s.handleCustomRecord(r.payload, r.addr)
		}
	}
}
```

## The Three-way Handshake

The three-way handshake establishes a secure connection between two parties. First, they use asymmetric encryption to exchange the symmetric encryption key. Then, for speed, they switch to symmetric encryption.

```

  Client                                  Server
    |                                       |
    |-- ClientHello (enc: server pubkey) -->|
    |<-------- HelloVerify (enc: client key)|
    |-- ClientHello (enc: server pubkey) -->|
    |<-------- ServerHello (enc: client key)|

```

```protobuf
   message Handshake {
     bytes session_ID = 1;
     bytes random    = 2;
     bytes cookie    = 3;
     bytes token     = 4;
     bytes key       = 5;
     int64 timestamp = 6;
   }
```

The handshake proceeds as follows:

1. **ClientHello** The client sends a `ClientHello` record encrypted with the server's public key, containing the client's symmetric key.

```go
func (c *ClientSocketManager) Connect() error {
	...
	clientHello.SetRandom(random)
	clientHello.SetKey(c.clientSymmKey)
	...
	payload, err = c.asymmCrypto.Encrypt(payload, c.serverAsymmPubKey)
	c.conn.Write(append([]byte{ClientHelloRecordType}, payload...))
	...
}
```

2. **HelloVerify** The server generates a cookie from the client's address and random, encrypts it with the client key, and sends it back. Saying verify who you are.

```go
// sayHelloVerify generates and sends a HelloVerify record to the client.
func (s *ServerSocketManager) sayHelloVerify(addr *net.UDPAddr, h socket_i.HandshakeRecord) {
	cookie := s.sessionManager.GetAddrCookieHMAC(addr, h.GetRandom())
	...
	helloVerify := s.encoder.NewHandshakeRecord()
	helloVerify.SetCookie(cookie)
	helloVerify.SetTimestamp(time.Now().UnixNano() / int64(time.Millisecond))

	helloVerifyPayload, err := s.encoder.MarshalHandshake(helloVerify)
	...

	helloVerifyPayload, err = s.symmCrypto.Encrypt(helloVerifyPayload, h.GetKey())
	...
    helloVerifyMessage := append([]byte{HelloVerifyRecordType}, helloVerifyPayload...)
	err = s.sendToAddr(addr, helloVerifyMessage)
}
```

3. **ClientHello (client)** The client re-sends the cookie (to prove address ownership) along with a token to verify identity, encrypted with the server's public key.

```go
func (c *ClientSocketManager) handleHelloVerifyRecord(record *record) {
	payload, err := c.symmCrypto.Decrypt(record.Body, c.clientSymmKey)
	helloVerify, err := c.encoder.UnmarshalHandshake(payload)

	clientHello.SetCookie(helloVerify.GetCookie())
	clientHello.SetToken(encryptedToken)
	...
	payload, err = c.asymmCrypto.Encrypt(payload, c.serverAsymmPubKey)
	c.conn.Write(append([]byte{ClientHelloRecordType}, payload...))
}
```

4. **ServerHello** The server validates the record and authenticates the token. If valid, it generates a session ID, saves the session, encrypts it, and sends it back.

```go

// sayServerHello processes the second client handshake and completes the handshake process.
func (s *ServerSocketManager) sayServerHello(addr *net.UDPAddr, h socket_i.HandshakeRecord) {
	cookie := s.sessionManager.GetAddrCookieHMAC(addr, h.GetRandom())
	if !s.HMAC.Compare(h.GetCookie(), cookie) { return }

	id, err := s.authenticator.Authenticate(h.GetToken())
	client, err := s.registerClient(addr, id, h.GetKey())
	...
	serverHello.SetSessionID(client.sessionID)
	s.sendToClient(client, ServerHelloRecordType, ...)
}
```

## Custom Records Handling

After the handshake, clients can send custom records. The server finds the client by address, decrypts the body, validates the session ID, and calls the registered handler.

```go
func (s *ServerSocketManager) handleCustomRecord(r *record, addr *net.UDPAddr) {
	cl, err := s.findClientWithAddr(addr)
	...
	payload, err := s.symmCrypto.Decrypt(r.Body, cl.eKey)
	...
	sessionID, body, err := splitSessionIDAndBody(payload, len(cl.sessionID))
	...
	if !bytes.Equal(sessionID, cl.sessionID) {
		...
		return
	}

	go s.onCustomClientRequest(cl.ID, r.Type, body)
}
```

On the client side, `SendToServer` prepends the session ID before encrypting and sending.

```go
func (c *ClientSocketManager) SendToServer(t byte, message []byte) error {
	payload := append(c.sessionID, message...)
	payload, err = c.symmCrypto.Encrypt(payload, c.clientSymmKey)
	_, err := c.conn.Write(append([]byte{t}, payload...))
	return err
}
```

## Ping Record Handling

The client sends periodic pings using a ticker. The server responds with a pong containing the received timestamp, and the client calculates the round-trip time.

```go
func (c *ClientSocketManager) requestPing() {
	for {
		select {
		case <-c.pingStopSignal:
			return
		case <-c.pingTicker.C:
			if len(c.sessionID) == 0 { continue }
			ping.SetSentAt(time.Now().UnixNano() / int64(time.Millisecond))
			c.SendToServer(PingRecordType, ...)
		}
	}
}
```

The server validates the session ID and sends a pong back. The client calculates latency from the timestamps.

```go
func (c *ClientSocketManager) handlePongRecord(record *record) {
	payload, err := c.symmCrypto.Decrypt(record.Body, c.clientSymmKey)
	pong, err := c.encoder.UnmarshalPong(payload)
	c.onPingResult(pong.GetReceivedAt() - pong.GetPingSentAt())
}
```

## Sending to a Client / Broadcasting to Certain Clients

The server can send to a single client by ID or broadcast to a list of clients after encrypting the payload with the client's symmetric key.

```go
func (s *ServerSocketManager) BroadcastToClients(clientIDs []uuid.UUID, typ byte, payload []byte) {
	for _, cl := range clientIDs {
		s.wg.Add(1)
		go func(c *client) {
			defer s.wg.Done()
			s.sendToClient(c, typ, payload)
		}(cl)
	}
}

func (s *ServerSocketManager) sendToClient(client *client, typ byte, payload []byte) error {
	payload, err = s.symmCrypto.Encrypt(payload, client.eKey)
	return s.sendToAddr(client.addr, append([]byte{typ}, payload...))
}
```

## Garbage Collection

If `heartbeatExpiration` is set, a ticker runs a cleanup routine that removes clients that haven't sent a ping within the expiration window.

```go
func (s *ServerSocketManager) clientGarbageCollection() {
	for {
		select {
		case <-s.garbageCollectionStop:
			return
		case <-s.garbageCollectionTicker.C:
			for _, c := range s.clients {
				if time.Now().After(c.lastHeartbeat.Add(s.heartbeatExpiration)) {
					s.clientsLock.Lock()
					delete(s.clients, c.ID)
					s.clientsLock.Unlock()
				}
			}
		}
	}
}
```

## Regrets

- Should have used `context.Context` for stop signals instead of `chan bool` — it would compose better with Go's cancellation patterns.
- `findClientWithAddr` does a linear scan over all clients to find by address. The initial design was for a max of 10 clients. A `map[string]*client` keyed by `addr.String()` would have been O(1).
- The garbage collection locks per-delete inside the range loop. Should collect expired IDs first, then delete in one lock.
