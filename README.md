# GGS AMP Templates

AMP (CubeCoders) deployment templates for Game Golem Studios' internal tooling.

Contains no secrets. The templates reference a private application repository, so
installing one requires a GitHub token with read access to it.

## Templates

- **Golem Control** (`golem-control.*`) — the studio's dedicated game server panel.

## Use

In AMP: **Configuration → Instance Deployment → Configuration Repositories**, add:

    Codewow/ggs-amp-templates:main

Then **Fetch** and refresh. New instances offer *Golem Control* under the `GGS` prefix.
