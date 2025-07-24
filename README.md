# Website

This website is built using [Docusaurus](https://docusaurus.io/), a modern static website generator.

## Installation

```bash
npm install
```

## Local Development

```bash
npm start
```

This command starts a local development server and opens up a browser window. Most changes are reflected live without having to restart the server.

## Build

```bash
npm run build
```

This command generates static content into the `build` directory and can be served using any static contents hosting service.

## Serve Locally

```bash
npm run serve
```

Serves the built website locally.

## Type Checking

```bash
npm run typecheck
```

Runs TypeScript type checking.

## Clear Cache

```bash
npm run clear
```

Clears the Docusaurus cache.

## Deployment

This project uses GitHub Actions for automatic deployment to GitHub Pages. When you push to the main branch, the site will be automatically built and deployed.

Manual deployment using Docusaurus:

```bash
npm run deploy
```

Or with environment variables:

```bash
GIT_USER=<Your GitHub username> npm run deploy
```
