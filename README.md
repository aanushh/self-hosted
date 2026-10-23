# self-hosted

Dictionary of my self-hosted apps.

## Applications

- [AdGuard Home](https://adguard.com/en/welcome.html) - DNS filtering for the homelab
- [BentoPDF](https://www.bentopdf.com/docs/) - PDF tools
- [Caddy](https://caddyserver.com/) - Web server and reverse proxy
- [Dockhand](https://dockhand.pro/) - Manage Docker stacks and containers
- [Homepage](https://gethomepage.dev/) - Homelab dashboard
- [Linkding](https://linkding.link/) - Bookmark manager
- [SilverBullet](https://silverbullet.md/) - Personal knowledge base
- [Uptime Kuma](https://uptime.kuma.pet/) - Service availability monitoring

## Architecture

```mermaid
flowchart TB
    Clients["Clients"]

    Clients -->|DNS| AdGuard["AdGuard Home"]
    Clients -->|HTTP/HTTPS| Caddy["Caddy"]

    subgraph Docker["Docker Host"]
        Caddy

        subgraph Bento["bentopdf network"]
            BentoPDF["BentoPDF"]
        end

        subgraph DockhandNet["dockhand network"]
            Dockhand["Dockhand"]
        end

        subgraph HomepageNet["homepage network"]
            Homepage["Homepage"]
        end

        subgraph LinkdingNet["linkding network"]
            Linkding["Linkding"]
        end

        subgraph Silver["silverbullet network"]
            SilverBullet["SilverBullet"]
        end

        subgraph AdGuardNet["adguard network"]
            AdGuard
        end

        subgraph UptimeNet["uptime-kuma network"]
            UptimeKuma["Uptime Kuma"]
        end
    end

    Caddy -->|reverse proxy| BentoPDF
    Caddy -->|reverse proxy| Dockhand
    Caddy -->|reverse proxy| Homepage
    Caddy -->|reverse proxy| Linkding
    Caddy -->|reverse proxy| SilverBullet
    Caddy -->|reverse proxy| AdGuard
    Caddy -->|reverse proxy| UptimeKuma
```
