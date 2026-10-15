# From Zero to Fullstack — Cheatsheets

**Learn → Practice → Document → Reuse.**

A practical, beginner-friendly knowledge base for learning **modern full-stack development** with JavaScript, React, Node.js, Express, MongoDB, Docker, and GitHub Actions.

This library is designed for two audiences:
- **First-time developers:** follow each chapter step by step, from tool installation to a working application.
- **Returning developers:** quickly find verified commands, patterns, templates, and troubleshooting guidance for new projects.

**Language:** clear technical English. **Formats:** editable Markdown and an accompanying reviewed PDF for each published guide.

> **Status:** This repository contains the curriculum and chapter indexes. The table distinguishes **published Markdown guides** from planned documentation. Drafts for Docker and GitHub Actions exist outside this repository and will be published only after review.

**Current progress:** **11/22 Markdown cheatsheets published**. Chapters 1 and 2 are fully documented in Markdown; Chapter 3 now has three published guides. PDF exports are available separately while repository PDF uploads remain pending.

## Roadmap: Chapters 1–5

| # | Chapter | Cheatsheet | Publication status |
| :-- | :-- | :-- | :-- |
| 01 | [Foundations](chapter-01-foundations/) | [Complete Developer Environment Setup](chapter-01-foundations/01-developer-environment/README.md) | **Published — Markdown** |
| 02 | [Foundations](chapter-01-foundations/) | [Git & GitHub from Scratch](chapter-01-foundations/02-git-github/README.md) | **Published — Markdown** |
| 03 | [Foundations](chapter-01-foundations/) | [React Project Setup with Vite](chapter-01-foundations/03-react-vite/README.md) | **Published — Markdown** |
| 04 | [Foundations](chapter-01-foundations/) | [ESLint & Prettier Configuration](chapter-01-foundations/04-eslint-prettier/README.md) | **Published — Markdown** |
| 05 | [Foundations](chapter-01-foundations/) | [Git Hooks with Husky & Commitlint](chapter-01-foundations/05-husky-commitlint/README.md) | **Published — Markdown** |
| 06 | [Backend Fundamentals](chapter-02-backend-fundamentals/) | [Node.js Fundamentals & Native HTTP Server](chapter-02-backend-fundamentals/06-node-http/README.md) | **Published — Markdown** |
| 07 | [Backend Fundamentals](chapter-02-backend-fundamentals/) | [Node.js File System & JSON Handling](chapter-02-backend-fundamentals/07-node-fs-json/README.md) | **Published — Markdown** |
| 08 | [Backend Fundamentals](chapter-02-backend-fundamentals/) | [MongoDB Installation & Native Node.js Driver](chapter-02-backend-fundamentals/08-mongodb-native-driver/README.md) | **Published — Markdown** |
| 09 | [Backend Development](chapter-03-backend-development/) | [Express.js REST API from Scratch](chapter-03-backend-development/09-express-rest-api/README.md) | **Published — Markdown** |
| 10 | [Backend Development](chapter-03-backend-development/) | [MongoDB with Mongoose](chapter-03-backend-development/10-mongoose/README.md) | **Published — Markdown** |
| 11 | [Backend Development](chapter-03-backend-development/) | [Building a Complete CRUD REST API](chapter-03-backend-development/11-crud-rest-api/README.md) | **Published — Markdown** |
| 12 | [Backend Development](chapter-03-backend-development/) | Backend Architecture: Routes, Services & Models | Planned |
| 13 | [Backend Development](chapter-03-backend-development/) | Automated Backend Testing with Jest | Planned |
| 14 | [Frontend Development](chapter-04-frontend-development/) | React Components, JSX, Props & State | Planned |
| 15 | [Frontend Development](chapter-04-frontend-development/) | Connecting React to a REST API | Planned |
| 16 | [Frontend Development](chapter-04-frontend-development/) | TanStack Query from Scratch | Planned |
| 17 | [Frontend Development](chapter-04-frontend-development/) | Complete React CRUD Application | Planned |
| 18 | [Frontend Development](chapter-04-frontend-development/) | Filtering, Sorting & Custom Hooks | Planned |
| 19 | [DevOps](chapter-05-devops/) | Docker Fundamentals & Installation | Planned |
| 20 | [DevOps](chapter-05-devops/) | Dockerizing a MERN Fullstack Application | Draft available; review pending |
| 21 | [DevOps](chapter-05-devops/) | GitHub Actions CI from Scratch | Draft available; review pending |
| 22 | [DevOps](chapter-05-devops/) | Continuous Deployment with Render | Planned |

[Chapter 6 — Authentication](chapter-06-authentication/) is **in progress**. The source repository contains signup with bcrypt; login and JWT work will be reflected in the documentation after implementation and testing are confirmed.

## Source projects

These cheatsheets are based on actual chapter repositories, not merely theoretical examples.

| Chapter | Project repository |
| :-- | :-- |
| 1 — Foundations | [ch1-foundations](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch1-foundations) |
| 2 — Backend Fundamentals | [ch2-backend-fundamentals](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch2-backend-fundamentals) |
| 3 — Backend Development | [ch3-backend-development](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch3-backend-development) |
| 4 — Frontend Development | [ch4-frontend-development](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch4-frontend-development) |
| 5 — DevOps | [ch5-devops](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch5-devops) |
| 6 — Authentication (ongoing) | [ch6-authentication](https://github.com/XMAN-MHD/from-zero-to-fullstack-ch6-authentication) |

## How to use a guide

1. Read its **purpose** and **prerequisites**.
2. Follow **installation from scratch** on your system.
3. Complete **configuration and verification**.
4. Understand the concepts and **build the example**.
5. Run the documented **tests** and compare the expected results.
6. Consult **troubleshooting** when something fails.
7. Save the **quick reference and reuse checklist** for the next project.

Each full cheatsheet will contain: objective, prerequisites, system requirements, installation, configuration, verification, conceptual explanation, architecture, implementation, testing, common errors, security, best practices, quick reference, reuse checklist, and source/version notes.

### Planned guide layout

```text
chapter-01-foundations/
├── README.md                  # Chapter index
├── 01-developer-environment/
│   ├── README.md              # Main editable cheatsheet
│   ├── cheatsheet.pdf          # Reviewed PDF export
│   └── examples/              # Optional reusable snippets
└── ...
```

Most chapter folders currently contain indexes. Published Markdown guides **01–03** are in `chapter-01-foundations/` under their topic folders. Their PDF exports are available separately but are **not yet hosted in this GitHub repository**.

## Accuracy and security policy

- Clearly distinguish **implemented and verified**, **studied but not committed**, and **future improvements**.
- Match referenced code, paths, tools, and commands to a specific project/version.
- Never publish `.env` files, passwords, API keys, tokens, or real database credentials.
- Provide safe placeholders and document expected environment variable names.
- Treat documentation as **living material**, updated when the projects change.

## Contributing and license

See [CONTRIBUTING.md](CONTRIBUTING.md) for editorial standards and the review checklist. This repository is distributed under the [MIT License](LICENSE).

---

*Build it, understand it, document it, reuse it.*
