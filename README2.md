# LOGGING and TROUBLESHOOTING

## OverView

Kubernetes clusters rely on API communication and are particularly affected by network issues. When troubleshooting, use standard Linux tools and procedures. If the affected Pod lacks a shell (like `bash`), deploy a similar Pod with a shell, such as `busybox`. Begin by examining DNS configuration files and using utilities like `dig`. For deeper analysis, you might need to install tools like `tcpdump`.

Monitoring is essential for managing large workloads. It involves collecting key metrics—CPU, memory, disk, and network usage—on nodes and applications. Kubernetes provides this via the Metrics Server, which exposes a standard API (e.g., `/apis/metrics.k8s.io/`) for use by components like autoscalers.

Centralized logging across all nodes is not built into Kubernetes by default. Tools like Fluentd can aggregate logs into a unified logging layer, making it easier to visualize and search for issues—especially when network troubleshooting does not reveal the root cause. Fluentd can be downloaded from its official website.

For a comprehensive solution that combines logging, monitoring, and alerting, consider Prometheus, a CNCF project. Prometheus provides a time-series database and integrates with Grafana for visualization and dashboards.