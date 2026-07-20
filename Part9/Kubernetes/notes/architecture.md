# Kubernetes Manifest Interactions

How the four YAML files in this directory relate to each other.

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'fontSize':'15px',
  'fontFamily':'Segoe UI, Helvetica, Arial, sans-serif',
  'textColor':'#0f172a',
  'primaryTextColor':'#0f172a',
  'secondaryTextColor':'#0f172a',
  'tertiaryTextColor':'#0f172a',
  'nodeTextColor':'#0f172a',
  'labelTextColor':'#0f172a',
  'clusterTextColor':'#0f172a',
  'titleColor':'#0f172a',
  'edgeLabelBackground':'#ffffff',
  'lineColor':'#334155',
  'clusterBkg':'#f1f5f9',
  'clusterBorder':'#64748b'
}}}%%
flowchart LR
  U(["User / browser"])

  subgraph SEC["dbhost-secret.yml"]
    S["Secret<br/>dbhost-secret<br/>dbhost-user, dbhost-pass,<br/>dbhost-database"]
  end

  subgraph CFG["dbhost-config.yml"]
    C["ConfigMap<br/>dbhost-config<br/>dbhost-url = dbhost-service"]
  end

  subgraph APP["app.yml"]
    AS["Service<br/>app-service<br/>NodePort 30000 &rarr; 3006"]
    AD["Deployment<br/>app-deployment<br/>replicas: 1"]
    AP["Pod app=app<br/>rcaliwag/al-app:v1<br/>containerPort 3006"]
    AD -->|creates| AP
    AS -.->|"selector app=app"| AP
  end

  subgraph DB["dbhost.yml"]
    DS["Service<br/>dbhost-service<br/>ClusterIP 3306"]
    DD["Deployment<br/>dbhost-deployment<br/>replicas: 1"]
    DP["Pod app=dbhost<br/>103mysql:latest<br/>containerPort 3306"]
    DD -->|creates| DP
    DS -.->|"selector app=dbhost"| DP
  end

  U ==>|"NodeIP:30000"| AS
  S -.->|"secretKeyRef<br/>DB_USER, DB_PASS, DB_NAME"| AP
  C -.->|"configMapKeyRef<br/>DB_HOST"| AP
  AP ==>|"dbhost-service:3306"| DS

  classDef secret  fill:#fee2e2,stroke:#dc2626,stroke-width:2px,color:#7f1d1d
  classDef config  fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#78350f
  classDef deploy  fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#1e3a8a
  classDef pod     fill:#ccfbf1,stroke:#0d9488,stroke-width:2px,color:#134e4a
  classDef svc     fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d
  classDef user    fill:#ede9fe,stroke:#7c3aed,stroke-width:2px,color:#4c1d95

  class S secret
  class C config
  class AD,DD deploy
  class AP,DP pod
  class AS,DS svc
  class U user

  style SEC fill:#fff5f5,stroke:#dc2626,stroke-width:2px,color:#7f1d1d
  style CFG fill:#fffbeb,stroke:#d97706,stroke-width:2px,color:#78350f
  style APP fill:#f0f7ff,stroke:#2563eb,stroke-width:2px,color:#1e3a8a
  style DB  fill:#f0fdf6,stroke:#16a34a,stroke-width:2px,color:#14532d

  linkStyle 0,2 stroke:#2563eb,stroke-width:2px
  linkStyle 1,3 stroke:#16a34a,stroke-width:2px
  linkStyle 4,7 stroke:#7c3aed,stroke-width:3px
  linkStyle 5 stroke:#dc2626,stroke-width:2px
  linkStyle 6 stroke:#d97706,stroke-width:2px
```

**Legend:** purple = live request traffic, blue = owns/creates, green = label selector,
red = secret injection, amber = config injection. Dotted edges are name-based references
resolved at runtime, solid edges are ownership.

## Coupling points

- `dbhost-secret.yml` and `dbhost-config.yml` are referenced only by name from `app.yml`
  (`secretKeyRef: dbhost-secret`, `configMapKeyRef: dbhost-config`). Nothing enforces ordering,
  so apply them before `app.yml` or the pod stalls in `CreateContainerConfigError`.
- The ConfigMap value `dbhost-url: dbhost-service` is the link between the two Deployments. It is
  the Service name from `dbhost.yml`, resolved through cluster DNS. Rename that Service and the
  app breaks silently.
- Services select pods by label, not by Deployment name. The `app: app` and `app: dbhost` labels in
  the pod templates are the actual glue.

## Apply order

```sh
kubectl apply -f dbhost-config.yml
kubectl apply -f dbhost-secret.yml
kubectl apply -f dbhost.yml
kubectl apply -f app.yml
```

## The four API object roles

Each file is a different `kind:` of Kubernetes API object. The course recap:

| Object | Role here |
|---|---|
| **ConfigMap** | Non-sensitive key-value data. The only value stored is the database URL. |
| **Secret** | Base64-encoded sensitive data: the Database username and password, plus the database name (kept here rather than the ConfigMap for extra security). |
| **Deployment Manifest** | The desired state of a Pod. `spec:` defines the Pod; `template:` configures the container within. |
| **Service** | How to connect to a Pod. `dbhost-service` for in-cluster access to the Database; `app-service` for browser access to the App. |
| **NodePort** | A Service type allowing inbound traffic from outside Kubernetes. Not strictly an API object of its own, but without it the app is sealed off from the outside world. |

Base64 in the Secret is **encoding, not encryption**. Values are decoded trivially; real encryption
has to be enabled as a feature inside etcd.

## Naming convention

Every object follows `<service name>-<API object kind>`, with the kind often shortened:

- `dbhost` + ConfigMap becomes `dbhost-config`
- `dbhost` + Secret becomes `dbhost-secret`
- `dbhost` + Service becomes `dbhost-service`

Keeping the service prefix consistent is what makes the whole set easy to filter once deployed.

---

See also: [kubernetes-architecture.md](kubernetes-architecture.md) for the cluster these objects
deploy into, [cluster-pods-containers.md](cluster-pods-containers.md) for the Pod and container
hierarchy, and [scaling-updates-rollbacks.md](scaling-updates-rollbacks.md) for changing them on a
live cluster.
