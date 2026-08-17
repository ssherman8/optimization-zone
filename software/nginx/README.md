# NGINX Optimization Guides

This section contains optimization guides for [NGINX](https://nginx.org/) on Intel hardware.

NGINX is the world's most popular webserver.  It is free and open source software, distributed under the terms of a simplified 2-clause BSD-like license.  Because a web tier spends much of its time on TLS handshakes and on compressing responses, it benefits from offloading that cryptography and compression work off the CPU cores and onto dedicated accelerators.

## Available Guides

| Guide | Description |
| --- | --- |
| [NGINX with Intel® QAT](QAT/README.md) | Offload TLS handshake cryptography and compression to Intel® QuickAssist Technology (Intel® QAT) using async-mode-nginx, including hardware and software prerequisites, configuration, and Connections Per Second (CPS) benchmark results. |

## References

asynch_mode_nginx: https://github.com/intel/asynch_mode_nginx

NGINX: https://nginx.org/
