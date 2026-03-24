# Contributing to Superkabe

Thanks for your interest in contributing to Superkabe.

## How to Contribute

### Reporting Bugs

Open an issue with:
- What you expected to happen
- What actually happened
- Steps to reproduce
- Your environment (Node version, OS, database)

### Suggesting Features

Open an issue with the `enhancement` label. Describe the use case — what problem does this solve for outbound teams?

### Pull Requests

1. Fork the repo
2. Create a feature branch (`git checkout -b feature/your-feature`)
3. Make your changes
4. Run the tests (`npm test`)
5. Commit with a clear message
6. Push to your fork and open a PR

### Code Style

- TypeScript strict mode
- Prisma for all database access (no raw SQL unless justified)
- All state changes go through `entityStateService` — never write to status fields directly
- Platform integrations must implement the `PlatformAdapter` interface
- Add audit logging for any action that modifies infrastructure state

### Adding a New Platform Adapter

The fastest way to contribute is adding support for a new sending platform:

1. Create `src/adapters/yourPlatformAdapter.ts`
2. Implement the `PlatformAdapter` interface (4 methods)
3. Register it in `src/adapters/platformRegistry.ts`
4. Add webhook parsing in `src/controllers/`
5. Add tests

### Areas We Need Help

- Additional sending platform adapters (Woodpecker, Saleshandy, Mailshake)
- Improved DNS health checking
- Dashboard UI improvements
- Documentation

## Code of Conduct

Be respectful. Focus on the work. We're building infrastructure, not arguing on the internet.
