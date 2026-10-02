# self-hosted

Dictionary of my self-hosted apps.

## List of my self-hosted apps

- [AdGuard](https://adguard.com/en/welcome.html) - DNS & Ad Blocker for my Home Lab
- [Caddy](https://caddyserver.com/) - Proxy server for my Home Lab
- [Karakeep](https://karakeep.app/) - Organize my bookmarks
- [Silverbullet](https://silverbullet.md/) - Personal knowledge base
- [BentoPDf](https://www.bentopdf.com/docs/) - PDF manipulation tool

## Architecture

```mermaid
flowchart TB
    Clients["Clients"]

    Clients -->|DNS| AdGuard["AdGuard Home"]
    Clients -->|HTTPS| Caddy["Caddy"]

    subgraph Docker["Docker Host"]
        Caddy

        subgraph Bento["bentopdf network"]
            BentoPDF["BentoPDF"]
        end

        subgraph Silver["silverbullet network"]
            SilverBullet["SilverBullet"]
        end

        subgraph AdGuardNet["adguard network"]
            AdGuard
        end

        subgraph Karakeep["karakeep network"]
            KarakeepWeb["Karakeep Web"]
        end

        subgraph KarakeepInternal["karakeep-internal network"]
            Chrome["Chrome"]
            Meilisearch["Meilisearch"]
        end
    end

    Caddy -->|reverse proxy| BentoPDF
    Caddy -->|reverse proxy| SilverBullet
    Caddy -->|reverse proxy| AdGuard
    Caddy -->|reverse proxy| KarakeepWeb

    KarakeepWeb --> Chrome
    KarakeepWeb --> Meilisearch
```
