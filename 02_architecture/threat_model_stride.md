# SecureBank: STRIDE Threat Model

## 1. DFD Legend & Scope
- **Process:** Rounded rectangle (handles logic)
- **Data Store:** Open rectangle (stores data)
- **External Entity:** Square (users/systems)
- **Trust Boundary:** Dashed line (where trust levels change)
- **Data Flow:** Arrow (data in transit)

## 2. Data Flow Diagram (DFD)
```mermaid
flowchart TD
    %% External Entities
    Customer[Customer]
    Employee[Branch Employee]
    Attacker[Attacker]

    %% Trust Boundaries
    subgraph Internet [Untrusted: Public Internet]
        Customer
        Employee
        Attacker
    end

    subgraph DMZ [Edge: DMZ]
        WAF[WAF / Reverse Proxy]
        LB[Load Balancer]
    end

    subgraph AppZone [Internal: Application Subnet]
        IAM[IAM Service]
        API[API Service]
    end

    subgraph DataZone [Internal: Private Data Subnet]
        DB[(PostgreSQL DB)]
    end

    subgraph SOC [SOC: Secure Terminal Zone]
        SIEM[SIEM / Audit Log Store]
        Analyst[SOC Analyst]
    end

    %% Flows (with Trust Crossing IDs)
    Customer -->|X1: HTTPS Login/Request| WAF
    Employee -->|X2: HTTPS VPN Request| WAF
    Attacker -.->|X3: Malicious Payload| WAF
    
    WAF -->|X4: Sanitized HTTP| LB
    LB -->|X5: Routed Request| API
    
    API -->|X6: Auth Query| IAM
    IAM -->|X7: JWT Token| API
    
    API -->|X8: SQL Query| DB
    DB -->|X9: Result Set| API
    
    API -->|X10: Telemetry/Logs| SIEM
    WAF -->|X11: Traffic Logs| SIEM
    SIEM -->|X12: Alerts| Analyst
