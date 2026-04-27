# Security

## Threat model

One-person homelab on a residential connection.

## Update 

- Edge components: patched promptly when CVEs land.
- Hypervisor and backup server: quarterly review, with security patches applied when needed.
- Application LXCs: rolling updates on a regular schedule. certain ones take precent
- Container images: re-pulled on the same rolling schedule.

## Backups

Hypervisor-level backups go to a dedicated backup server. Conservative retentions and backups are verified periofically.The rebuild order is documented. 

## Limitations

This is a learning environment.

- No High Availability - One hypervisor, one firewall
- One-person ops
