# Domain Name System (DNS)

DNS is the backbone of the internet, translating human-readable domain names into IP addresses that computers can understand. Understanding DNS is crucial for system design, performance optimization, and troubleshooting.

## What is DNS?

### Definition
The Domain Name System (DNS) is a hierarchical, distributed naming system that translates domain names (like google.com) into IP addresses (like 142.250.190.78).

### Why DNS Matters
- **Human-Friendly**: People remember names, computers use numbers
- **Scalability**: Distributed system handles billions of lookups daily
- **Flexibility**: Easy to change IP addresses without user impact
- **Load Balancing**: Direct traffic to different servers based on rules

## DNS Hierarchy

### Root Level
- **13 Root Servers**: Managed by 12 organizations worldwide
- **Notation**: A through M (a.root-servers.net through m.root-servers.net)
- **Purpose**: Direct queries to appropriate Top-Level Domain (TLD) servers

### Top-Level Domains (TLDs)
- **Generic TLDs**: .com, .org, .net, .info, .biz
- **Country Code TLDs**: .us, .uk, .de, .jp, .in
- **Sponsored TLDs**: .edu, .gov, .mil, .travel
- **Special TLDs**: .localhost, .test

### Second-Level Domains
- **User-Registered**: google.com, amazon.com, facebook.com
- **Subdomains**: mail.google.com, api.github.com

### DNS Resolution Process

#### Recursive Resolution (Client → Resolver)
1. **Client Query**: User types google.com in browser
2. **Local DNS**: Check local cache first
3. **Root Server**: If not cached, query root server
4. **TLD Server**: Get authoritative server for .com
5. **Authoritative Server**: Get IP for google.com
6. **Response**: IP address returned to client

#### Iterative Resolution (Resolver → Authoritative)
Resolver queries each level directly instead of relying on recursion.

## DNS Record Types

### A Record (Address)
Maps domain name to IPv4 address.
```
google.com    IN  A     142.250.190.78
```

### AAAA Record (IPv6 Address)
Maps domain name to IPv6 address.
```
google.com    IN  AAAA  2a00:1450:4001:814::200e
```

### CNAME Record (Canonical Name)
Creates alias for another domain name.
```
www.google.com    IN  CNAME   google.com
```

### MX Record (Mail Exchange)
Specifies mail servers for domain.
```
gmail.com    IN  MX  10    alt1.gmail-smtp-in.l.google.com
```

### TXT Record (Text)
Stores arbitrary text data (SPF, DKIM, verification).
```
google.com    IN  TXT     "v=spf1 include:_spf.google.com ~all"
```

### NS Record (Name Server)
Delegates subdomain to different name servers.
```
google.com    IN  NS      ns1.google.com
```

### SOA Record (Start of Authority)
Contains administrative information about zone.
```
google.com    IN  SOA     ns1.google.com admin.google.com (
                             2023120101 ; serial
                             7200       ; refresh
                             3600       ; retry
                             1209600    ; expire
                             300        ; minimum TTL
                         )
```

## DNS Caching

### Types of DNS Caches

#### Browser DNS Cache
- Stores DNS lookups for browser session
- Fastest access (microseconds)
- Cleared when browser closes

#### OS DNS Cache
- System-level cache (hosts file, DNS client)
- Persists across browser sessions
- Managed by operating system

#### Recursive Resolver Cache
- ISP or public DNS server cache
- Shared across many users
- Configurable TTL (Time To Live)

#### Authoritative Server Cache
- Internal caching at DNS servers
- Reduces load on backend databases

### TTL (Time To Live)
- **Definition**: How long DNS record can be cached
- **Unit**: Seconds (typically 300-86400)
- **Trade-offs**: Lower TTL = faster changes but more queries
- **Best Practices**: 300s for development, 3600s+ for production

## DNS Load Balancing

### Round Robin DNS
Multiple A/AAAA records for same domain.
```
api.example.com    IN  A     192.168.1.1
api.example.com    IN  A     192.168.1.2
api.example.com    IN  A     192.168.1.3
```

**Advantages:**
- Simple implementation
- No special infrastructure needed
- Client-side load distribution

**Disadvantages:**
- No health checking
- Uneven load if TTLs vary
- No session persistence

### Geographic DNS (GeoDNS)
Routes users to nearest server based on location.

**Implementation:**
- **EDNS Client Subnet**: DNS server sees client IP
- **GeoIP Database**: Maps IP to geographic location
- **Different Responses**: Return different IPs based on location

**Example:**
```
# US users → us-east-1.api.example.com (A: 1.2.3.4)
# EU users → eu-west-1.api.example.com (A: 5.6.7.8)
# ASIA users → ap-south-1.api.example.com (A: 9.10.11.12)
```

### Weighted DNS
Assign different weights to servers.

**Implementation:**
- Multiple records with different priorities
- DNS server returns based on weights
- Can be combined with health checks

## DNS Security

### DNSSEC (DNS Security Extensions)
- **Purpose**: Prevent DNS spoofing and cache poisoning
- **How it Works**: Cryptographically signs DNS records
- **Key Components**: 
  - **RRSIG**: Digital signature for records
  - **DNSKEY**: Public key for verification
  - **DS**: Delegation signer record

### DNS over HTTPS (DoH)
- **Purpose**: Encrypt DNS queries to prevent eavesdropping
- **How it Works**: DNS queries over HTTPS instead of UDP
- **Benefits**: Privacy, prevents DNS hijacking
- **Adoption**: Supported by major browsers and resolvers

### DNS over TLS (DoT)
Similar to DoH but uses TLS instead of HTTPS.

## DNS in System Design

### CDN Integration
- **DNS Steering**: Route users to optimal CDN edge location
- **Failover**: Automatic switching to backup CDN if primary fails
- **Load Balancing**: Distribute traffic across multiple CDN providers

### Microservices DNS
- **Service Discovery**: DNS for locating microservices
- **Internal DNS**: Private DNS zones for service-to-service communication
- **Health Checks**: DNS-based service health monitoring

### Multi-Region Deployment
- **Latency-Based Routing**: Route to closest region
- **Failover Routing**: Automatic failover to healthy regions
- **Weighted Routing**: Distribute load based on capacity

## DNS Best Practices

### Performance
- **Minimize TTL**: For frequently changing records
- **Use CDN**: Reduce DNS lookup latency
- **Pre-resolve**: Browser pre-resolves likely domains
- **Optimize Records**: Keep responses small

### Reliability
- **Multiple NS Records**: At least 2-3 name servers
- **Diverse Locations**: Name servers in different geographic regions
- **Monitoring**: Track DNS resolution times and failures
- **Backup Resolvers**: Fallback DNS servers

### Security
- **Enable DNSSEC**: For critical domains
- **Use DoH/DoT**: For privacy and security
- **Regular Audits**: Monitor DNS configurations
- **Access Controls**: Limit who can modify DNS records

## DNS Troubleshooting

### Common Issues

#### DNS Resolution Failures
- **NXDOMAIN**: Domain doesn't exist
- **SERVFAIL**: Server error
- **TIMEOUT**: Query timeout
- **REFUSED**: Query refused

#### Tools for Debugging
```bash
# Check DNS resolution
dig google.com

# Trace DNS resolution path
dig +trace google.com

# Check specific record type
dig TXT google.com

# Use specific DNS server
dig @8.8.8.8 google.com

# Check DNS cache
sudo killall -HUP mDNSResponder  # macOS
sudo systemctl restart systemd-resolved  # Linux
```

#### DNS Propagation
- **Time**: 24-48 hours for global propagation
- **Factors**: TTL values, ISP caching, CDN purging
- **Monitoring**: Use tools like DNS Checker, WhatsMyDNS

## DNS in Java Applications

### DNS Lookup in Java
```java
import java.net.InetAddress;

public class DNSLookup {
    public static void main(String[] args) throws Exception {
        InetAddress address = InetAddress.getByName("google.com");
        System.out.println("IP Address: " + address.getHostAddress());
        
        // Reverse lookup
        InetAddress reverse = InetAddress.getByName("142.250.190.78");
        System.out.println("Host Name: " + reverse.getHostName());
    }
}
```

### DNS Caching in Java
```java
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.security.Security;

public class DNSCacheConfig {
    public static void main(String[] args) {
        // Set DNS cache TTL
        Security.setProperty("networkaddress.cache.ttl", "60");  // 60 seconds
        Security.setProperty("networkaddress.cache.negative.ttl", "10");  // 10 seconds
        
        // Disable DNS caching
        Security.setProperty("networkaddress.cache.ttl", "0");
        Security.setProperty("networkaddress.cache.negative.ttl", "0");
    }
}
```

### Custom DNS Resolver
```java
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.Arrays;

public class CustomDNSResolver {
    
    private static final String[] DNS_SERVERS = {
        "8.8.8.8",      // Google DNS
        "1.1.1.1",      // Cloudflare DNS
        "208.67.222.222" // OpenDNS
    };
    
    public static InetAddress resolve(String hostname) throws UnknownHostException {
        for (String dnsServer : DNS_SERVERS) {
            try {
                // Custom DNS resolution logic would go here
                // This is simplified - actual implementation would use DNS protocol
                return InetAddress.getByName(hostname);
            } catch (UnknownHostException e) {
                // Try next DNS server
                continue;
            }
        }
        throw new UnknownHostException("Failed to resolve: " + hostname);
    }
}
```

## Advanced DNS Concepts

### Anycast DNS
- **Concept**: Multiple servers share same IP address
- **Benefit**: Requests routed to nearest server
- **Use Case**: Root DNS servers, CDN DNS

### DNS-Based Load Balancing
- **Global Server Load Balancing (GSLB)**
- **Geographic Load Balancing**
- **Health-Checked Load Balancing**

### DNS as a Service (DNSaaS)
- **Managed DNS**: Cloud providers manage DNS infrastructure
- **Advanced Features**: DDoS protection, advanced routing
- **Examples**: AWS Route 53, Google Cloud DNS, Azure DNS

## Conclusion

DNS is a critical component of internet infrastructure that most users never think about but powers every web request. Understanding DNS fundamentals, caching, load balancing, and security is essential for designing scalable, reliable systems.

**Key Takeaways:**
- DNS translates names to IP addresses
- Caching improves performance but affects change propagation
- Load balancing distributes traffic geographically
- Security features like DNSSEC prevent attacks
- Monitoring and troubleshooting are crucial for reliability

DNS decisions impact system performance, reliability, and user experience - choose wisely!
