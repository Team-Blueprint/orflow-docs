# Orflow Documentation

Documentation site for [Orflow](https://orflow.io) — a recurring billing API built on Nomba. Built with [Mintlify](https://mintlify.com).

## Contents

- **Quickstart** — get up and running in minutes
- **Core Concepts** — plans, subscriptions, customers, webhooks
- **Guides** — integration walkthroughs and best practices
- **API Reference** — auto-generated OpenAPI reference for every endpoint

## Development

```bash
npm i -g mint
mint dev
```

The local preview runs at `http://localhost:3000`.

## Publishing

Changes are deployed to production automatically after pushing to the default branch. Install the [Mintlify GitHub App](https://dashboard.mintlify.com/settings/organization/github-app) to enable auto-deployment.

## Structure

```
docs/
├── api-reference/     # OpenAPI-generated endpoint docs
├── core-concepts/     # Plans, subscriptions, customers, webhooks
├── guides/            # Integration guides
├── index.mdx          # Landing page
├── quickstart.mdx     # Quickstart guide
├── docs.json          # Mintlify configuration
└── styles.css         # Custom styling
```

## Troubleshooting

- Run `mint update` to update the CLI.
- If a page returns 404, verify it's listed in `docs.json`.
