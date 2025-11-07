flowchart TD
  %% === Entry Points ===
  A[CI/CD Pipeline or Local Runner<br/>Selenium RemoteWebDriver] -->|Create session / WebDriver commands| R[Grid Router]

  %% === Core Grid Control Plane ===
  subgraph G[Selenium Grid Control Plane]
    direction TB
    R -->|New session request| Q[Session Queue]
    R -->|Lookup active session| M[Session Map]
    Q -->|Dequeues request| D[Distributor]
    D -->|Registers/updates sessions| M
  end

  %% === Browser Nodes on Kubernetes ===
  subgraph K8s[Kubernetes Cluster]
    direction TB
    subgraph BN[Selenium Browser Nodes (Pods/Deployments)]
      direction LR
      N1[(Chrome Nodes)]
      N2[(Firefox Nodes)]
      N3[(Edge Nodes)]
      N4[(Opera Nodes)]
      N5[(Other Browsers)]
    end

    %% Distributor selects best-fit node based on capacity/availability
    D -->|Selects best-fit node| N1
    D -->|Selects best-fit node| N2
    D -->|Selects best-fit node| N3
    D -->|Selects best-fit node| N4
    D -->|Selects best-fit node| N5

    %% Nodes establish/maintain sessions (tracked in Session Map)
    N1 -->|Start/maintain session| M
    N2 -->|Start/maintain session| M
    N3 -->|Start/maintain session| M
    N4 -->|Start/maintain session| M
    N5 -->|Start/maintain session| M
  end

  %% === Command/Response Path ===
  A <-->|HTTP/WebDriver traffic| R
  R <-->|Route to specific node| N1
  R <-->|Route to specific node| N2
  R <-->|Route to specific node| N3
  R <-->|Route to specific node| N4
  R <-->|Route to specific node| N5

  %% === Event-Driven Autoscaling with KEDA ===
  subgraph AS[Event-Driven Autoscaling]
    direction TB
    K[KEDA<br/>(Event-Driven Autoscaler)]
    H[HPA managed by KEDA<br/>(per browser deployment)]
    CA[Cluster Autoscaler<br/>(Worker Node Provisioner)]
  end

  %% KEDA watches load signals (e.g., queue length, pending sessions)
  Q -->|Queue length / pending sessions| K
  K -->|Set target replicas| H
  H -->|Scale Pods (per browser type)| N1
  H -->|Scale Pods (per browser type)| N2
  H -->|Scale Pods (per browser type)| N3
  H -->|Scale Pods (per browser type)| N4
  H -->|Scale Pods (per browser type)| N5

  %% If pods are pending, Cluster Autoscaler adds nodes
  H -->|If pods pending| CA
  CA -->|Provision/scale worker nodes| K8s

  %% === Resilience & Redundancy Notes ===
  classDef note fill:#f7f7f7,stroke:#bbb,color:#333,font-size:12px;
  X1([If a node becomes unavailable,<br/>Distributor re-queues and selects another node]):::note
  X2([Session Map ensures correct routing<br/>of subsequent commands to the right node]):::note
  X3([KEDA scales down during low demand,<br/>Cluster Autoscaler downsizes nodes safely]):::note

  D -.on failure.-> Q
  M -.guides routing.-> R
  H -.scale down targets.-> BN
  CA -.scale in nodes.-> K8s
