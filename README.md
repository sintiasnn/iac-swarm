# iac-swarm

## 🏁 Quick Start

### Prerequisites

To get started, install Moon globally [(see documentation)][moon-install] and ensure you have
`Docker >=20.10` and `Moon >=1.32` installed on your machine.

### Up and running

1. Setup the monorepo environment: `moon setup`
2. Create `.env` file or duplicate the `.env.example` file, then configure required variables.
3. Generate application secret key: `pnpm generate:key`
4. Run the database develoment server: `pnpm compose:up`
5. Launch the development servers: `moon run '#apps:dev'` or `moon :dev`

After all, it will simultaneously start the backend and website applications.

Strapi backend(cms) and website app will run simultaneously in development mode:

- Backend application will run at: <http://localhost:8000>
- Website application will run at: <http://localhost:3000>

Refer to the Moon [command reference](#moon-commands) below for more options.

### Moon commands

After setting up your project, use the following commands for common tasks:

| Command                 | Description                              |
|-------------------------|------------------------------------------|
| `moon :dev`             | Start developing the project             |
| `moon :build`           | Build all projects                       |
| `moon :test`            | Run tests in all projects                |
| `moon :lint`            | Lint code in all projects                |
| `moon :format`          | Format code in all projects              |
| `moon <project>:<task>` | Run specific task by project             |
| `moon check <project>`  | Run check for individual project         |
| `moon check --all`      | Run check for all tasks                  |
| `moon run '#tag:task'`  | Run a task in all projects with a tag    |
| `moon project-graph`    | Display an interactive graph of projects |

Type `moon help` for more information.
Refer to the [moon tasks documentation](https://moonrepo.dev/docs/run-task) for more details.

[moonrepo]: https://moonrepo.dev/

### Extras

This project includes a database management tool ([pgweb](https://sosedoff.github.io/pgweb/))
and a local SMTP server ([mailpit](https://mailpit.axllent.org/)) to support development.

- Pgweb address: <http://localhost:54321>
- Mailpit address: <http://localhost:8025>

## 📦 Managing Dependencies

To add a new dependency to a project, you can use the following command:

```sh
pnpm --filter <project> add <dependency>
```

Or, if you want to add development dependencies, you can use the following command:

```sh
pnpm --filter <project> add -D <dependency>
```

Example:

```sh
pnpm --filter web add -D vitest
```

### Updating dependencies

To update all projects dependencies, you can use the following command:

```sh
pnpm run update-deps
```

### Cleanup Projects

Sometimes it is necessary to clean up dependencies and build artifacts from the project.
To do this, you can use the following command:

```sh
pnpm run cleanup
```

After all, you can reinstall the dependencies and build the project.

## 🐳 Docker Container

### Development Server

```sh
# Start development server
docker-compose up -d

# Stop development server
docker-compose down --remove-orphans --volumes
```

## 🚀 Deployment

Read [Deployment Guide](./DEPLOY.md) for detailed documentation.

<!-- link reference definition -->
[nodejs]: https://nodejs.org/en/download
[docker]: https://docs.docker.com/engine/install
[moon-install]: https://moonrepo.dev/docs/install
