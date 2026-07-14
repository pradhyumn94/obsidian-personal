![[Excalidraw/Networking 101]]



Topics to read up on 
- [ ] QUIC 
- [ ] Webhooks and SSE
- [ ] WebRTC (I know this just a refresh needed)
- [ ] CFRDT (Conflict free resolution data types)
- [ ] Layer-4 vs layer-7 load balancer


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

Types of load balancers
- **Hardware Load Balancers**: Physical devices like F5 Networks BIG-IP
- **Software Load Balancers**: HAProxy, NGINX, Envoy
- **Cloud Load Balancers**: AWS ELB/ALB/NLB, Google Cloud Load Balancing, Azure Load Balancer
