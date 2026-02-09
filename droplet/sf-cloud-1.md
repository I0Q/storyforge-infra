# sf-cloud-1 (cloud connector droplet)

- Purpose: VPC-only gateway for Tinybox compute node over Tailscale.
- Public IP: 159.65.251.41
- Private IP (VPC): 10.108.0.3
- Tailscale IP: 100.95.125.10

Services:
- tinybox-compute-node-gateway.service
  - live code: /opt/tinybox-compute-node-gateway/app/main.py
  - repo: https://github.com/I0Q/tinybox-compute-node-gateway

Tokens (on droplet):
- /root/.config/storyforge-cloud/gateway_token  (auth between App Platform and gateway)
- /root/.config/storyforge-cloud/tinybox_token  (auth between gateway and Tinybox)
