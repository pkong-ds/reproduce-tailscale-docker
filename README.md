# Demonstrating Tailscale Exit Node Failure

## TL;DR

This line `sysctl -w net.ipv4.conf.eth0.rp_filter=2
` fixes the exit-node issues

## Next Step

- Check if this change is secure to do on k8s pod
- Investigate how to change that command in k8s
  - [k8s doc on safe-and-unsafe-sysctls](https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/#safe-and-unsafe-sysctls)

## Description

This document outlines a reproducible issue where Tailscale network connectivity fails after setting an exit node within a Docker container. Specifically, the IPv4 address resolution and network traffic are disrupted.

## Steps to Reproduce

1.  **Start the Docker container:**
    ```bash
    docker compose up -d
    ```

2.  **Enter the Tailscale container's shell:**
    ```bash
    docker compose exec -it tailscale sh
    ```

3.  **Install `curl`:**
    ```bash
    apk update && apk add curl
    ```

4.  **Verify initial network connectivity:**
    ```bash
    tailscale netcheck
    ```
    * **Expected:** Output shows a valid IPv4 address.

5.  **Test internet connectivity:**
    ```bash
    ping google.com
    ```
    * **Expected:** Successful ping with normal data packet transfer.

6.  **Set the Tailscale exit node to `eu.oursky.cloud`:**
    ```bash
    tailscale set --exit-node=100.64.0.20 --exit-node-allow-lan-access
    ```

7.  **Verify network connectivity again:**
    ```bash
    tailscale netcheck
    ```
    * **Observed:** The command hangs for approximately 3 seconds and then returns the following, indicating an IPv4 address resolution failure:
        ```
        Report:
          * Time: 2025-03-26T13:06:35.272378838Z
          * UDP: false
          * IPv4: (no addr found)
          * IPv6: no, unavailable in OS
          * MappingVariesByDestIP:
          * PortMapping:
          * CaptivePortal: false
          * Nearest DERP: unknown (no response to latency probes)
        ```

8.  **Test internet connectivity again:**
    ```bash
    ping google.com
    ```
    * **Observed:** The command hangs for approximately 10 seconds and then returns:
        ```
        ping: google.com: Try again
        ```

## Expected Result

Steps 7 and 8 should produce the same successful network connectivity results as steps 4 and 5, respectively.

## Solution

The issue can be resolved by adjusting the reverse path filtering mode for the network interface `eth0` within the Docker container.

1.  **Set the reverse path filtering mode to loose mode:**
    ```bash
    # Requires privileged access (see docker-compose.yaml#L16)
    sysctl -w net.ipv4.conf.eth0.rp_filter=2
    ```

2.  **Verify network connectivity after the change:**
    ```bash
    tailscale netcheck
    ```
    * **Expected:** Output shows a valid IPv4 address.

3.  **Test internet connectivity:**
    ```bash
    ping google.com
    ```
    * **Expected:** Successful ping with normal data packet transfer.

4.  **Confirm the exit node's public IP address:**
    ```bash
    curl ifconfig.me
    ```
    * **Expected:** Returns `65.108.15.186`, which [is the public IP address of `eu.oursky.cloud`.](https://github.com/oursky/oursky.cloud/blob/aa90d9c78d5129dba475c2aa81a995b9c573615e/eu.oursky.cloud/README.md?plain=1#L12)

5.  **Test setting another exit node (e.g., `hk.oursky.cloud`):**
    ```bash
    tailscale set --exit-node=100.64.0.43
    ```

6.  **Verify the new exit node's public IP address:**
    ```bash
    curl ifconfig.me
    ```
    * **Expected:** Returns `94.190.220.240`, [which is the public IP address of `hk.oursky.cloud`.](https://github.com/oursky/oursky.cloud/blob/aa90d9c78d5129dba475c2aa81a995b9c573615e/hk.oursky.cloud.md?plain=1#L7)

## Additional Context

This issue is related to a reported problem in the Tailscale GitHub repository:

* [tailscale/tailscale#3310](https://github.com/tailscale/tailscale/issues/3310)
