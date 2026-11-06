# zenith-docs

Public user documentation for the Zenith multi-cloud GPU compute platform.

This repository holds the **user-facing** help content only. It is consumed by the
private Zenith application as a Git submodule checked out at `help/docs/`, and is
rendered by VitePress as the help center served from `/docs/`.

## What lives here

- `index.md` — help center home page
- `guide/`, `compute/`, `workloads/`, `data/`, `projects/`, `account/`, `assets/`,
  `resources/`, `reference/`, `troubleshooting/`, `admin/` — user and operations guides
- `public/` — images, logos, and font license texts referenced by the pages

## What does not live here

Developer-facing material — architecture notes, protocol contracts, database and
ORM conventions, CI/CD and deployment topology — stays in the private Zenith
repository under `docs/`. Nothing from that tree is published here.

The VitePress configuration, theme, and self-hosted typeface also stay in the
private repository at `help/.vitepress/`, so this repository remains a plain
Markdown and asset checkout.

## Writing conventions

- Pages are written in Simplified Chinese, matching the existing pages in tone and
  structure. Source-level code comments inside code blocks stay in English.
- Every page starts with a `title` and `description` front matter block.
- Links between pages use root-relative paths such as
  `/compute/instances.html`. Do not link to the private `docs/` tree.
- Never commit credentials, API tokens, private keys, internal hostnames, or
  unmasked provider identifiers. Use placeholders such as `zenith.example.com`.

## Contributing

Changes land through a pull request against `main`:

1. Branch from the latest `main` using a semantic prefix such as `feature/`,
   `fix/`, or `docs/`.
2. Commit your edits on that branch.
3. Open a pull request against `main`. The default branching rules require a pull
   request and a linear history; merge with **Squash and merge**.

To pick the change up in the Zenith application, a maintainer there updates the
submodule pointer in a separate pull request against `dev`.

## Local preview

The site is built by the parent repository, not from this checkout alone. To preview
page content, clone Zenith and run its help center:

```bash
git clone --recurse-submodules https://github.com/bjzgcai/Zenith.git
cd Zenith/help
pnpm install --frozen-lockfile
pnpm dev
```

## License

Documentation text and images are released under the MIT License; see [LICENSE](LICENSE).
Bundled font license texts under `public/font-licenses/` remain under their own terms.
