![[Excalidraw/Networking 101]]

Topics to read up on
- [ ] QUIC
- [ ] Webhooks and SSE
- [ ] WebRTC (I know this just a refresh needed)
- [ ] CFRDT (Conflict free resolution data types)
- [ ] Layer-4 vs layer-7 load balancer
- [ ] **retry with exponential backoff** (jitter) https://builder.aws.com/content/3EumjoZascWd1oZiEgL8ORlv3qE/timeouts-retries-and-backoff-with-jitter

#### Key Characteristics of TCP

1. **Connection-oriented**: Establishes a dedicated connection before data transfer
2. **Reliable delivery**: Guarantees that data arrives in order and without errors
3. **Flow control**: Prevents overwhelming receivers with too much data
4. **Congestion control**: Adapts to network congestion to prevent collapse

Several options are available with most load balancers:
- **Round Robin**: Requests are distributed sequentially across servers
- **Random**: Requests are distributed randomly across servers
- **Least Connections**: Requests go to the server with the fewest active connections
- **Least Response Time**: Requests go to the server with the fastest response time
- **IP Hash**: Client IP determines which server receives the request (useful for session persistence)

#### Types of load balancers

- **Hardware Load Balancers**: Physical devices like F5 Networks BIG-IP
- **Software Load Balancers**: HAProxy, NGINX, Envoy
- **Cloud Load Balancers**: AWS ELB/ALB/NLB, Google Cloud Load Balancing, Azure Load Balancer

#### Circuit Breaker

1. The circuit breaker monitors for failures when calling external services
2. When failures exceed a threshold, the circuit "trips" to an open state
3. While open, requests immediately fail without attempting the actual call
4. After a timeout period, the circuit transitions to a "half-open" state
5. A test request determines whether to close the circuit or keep it open

- Fail Fast: Quickly reject requests to failing services instead of waiting for timeouts
- Reduce Load: Prevent overwhelming already struggling services with more requests
- Self-Healing: Automatically test recovery without full traffic load
- Improved User Experience: Provide fast fallbacks instead of hanging UI
- System Stability: Prevent failures in one service from affecting the entire system
