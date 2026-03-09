# Flutter Riverpod Feature-First MVVM Skill

> Flutter feature-first MVVM patterns with Riverpod 3.x codegen, Freezed 3.x sealed classes, GoRouter, Drift, and Hive CE persistence.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Flutter](https://img.shields.io/badge/Flutter-3.x-02569B.svg)](https://flutter.dev)
[![Riverpod](https://img.shields.io/badge/Riverpod-3.x-00B4D8.svg)](https://riverpod.dev)

> **Disclaimer:** This is an unofficial community resource. It is not affiliated with, endorsed by, or sponsored by Google, the Flutter team, or the Riverpod maintainers. "Flutter" is a trademark of Google LLC. "Riverpod" is maintained by Remi Rousselet.

## Installation

```bash
npx skills add akhlashashmi/flutter-apps-skill
```

Or manually clone into `~/.claude/skills/`:

```bash
git clone https://github.com/akhlashashmi/flutter-apps-skill ~/.claude/skills/flutter-apps-skill
```

## What's Included

This skill provides AI agents with guidance for Flutter development using a feature-first MVVM architecture and modern Flutter best practices.

### Core Stack

| Package | Version | Purpose |
|---------|---------|---------|
| flutter_riverpod | 3.2.1+ | State management |
| riverpod_annotation | 3.x | Codegen annotations |
| riverpod_generator | 3.x | Provider code generation |
| freezed | 3.2.5+ | Immutable data classes, unions |
| go_router | 17.1.0+ | Declarative routing |
| drift | 2.32.0+ | SQLite ORM and reactive queries |
| drift_dev | 2.32.0+ | Drift codegen and migration tooling |
| drift_flutter | 0.3.0+ | Flutter Drift connection helpers |
| hive_ce | 2.19.3+ | Binary local persistence |

### Architecture

Google-aligned feature-first MVVM structure:

- Organize code by feature first.
- Keep `presentation/` as the MVVM UI layer.
- Treat screens and widgets as the View.
- Treat Riverpod notifiers as the ViewModel.
- Keep `domain/` for pure business entities.
- Keep `data/` for models and datasources.
- Keep `repositories/` for feature data orchestration.

```text
lib/
|-- core/           # Shared: theme, utils, widgets, navigation, services
|-- features/       # Feature modules (auth, products, home, ...)
|   `-- feature_x/
|       |-- data/           # Models, datasources
|       |-- domain/         # Entities (pure Dart)
|       |-- repositories/   # Data orchestration
|       `-- presentation/   # Notifiers, screens, widgets
`-- main.dart
```

### Key Patterns

- **Feature-first MVVM** - Organize code by feature, then split it into `data`, `domain`, `repositories`, and `presentation`
- **Presentation as MVVM** - Screens/widgets are the View, Riverpod notifiers are the ViewModel
- **Codegen-only providers** - No `StateProvider`, `StateNotifierProvider`, or legacy providers
- **Sealed classes** - Exhaustive pattern matching with Dart's native `switch`
- **No prop drilling** - Child widgets watch providers directly
- **Async safety** - `if (!ref.mounted) return;` guards after every `await`
- **Unified Ref** - Single `Ref` type (no `AutoDisposeRef`, `ExampleRef`)

## Reference Files

| Topic | File |
|-------|------|
| Architecture layers | [architecture.md](references/architecture.md) |
| Atomic design (tokens to pages) | [atomic-design.md](references/atomic-design.md) |
| Riverpod 3.x codegen | [riverpod-codegen.md](references/riverpod-codegen.md) |
| Freezed 3.x sealed classes | [freezed-sealed.md](references/freezed-sealed.md) |
| State management patterns | [state-management.md](references/state-management.md) |
| Pagination, search, forms | [common-patterns.md](references/common-patterns.md) |
| Performance optimization | [performance.md](references/performance.md) |
| Flutter optimizations | [flutter-optimizations.md](references/flutter-optimizations.md) |
| Extensions and utilities | [extensions-utilities.md](references/extensions-utilities.md) |
| Hive CE persistence, TypeAdapters | [hive-persistence.md](references/hive-persistence.md) |
| Drift persistence, scalable structure, migrations | [drift-persistence.md](references/drift-persistence.md) |

## Compatible Agents

- [Claude Code](https://code.claude.com/)
- [Cursor](https://cursor.sh/)
- [Windsurf](https://windsurf.ai/)
- Any agent supporting the [Agent Skills](https://agentskills.io/) standard

## Usage

Once installed, the skill automatically activates when you:

- Build, review, or refactor Flutter apps
- Work with Riverpod state management
- Implement Freezed data classes
- Set up GoRouter navigation

Or invoke directly:

```text
/flutter-apps-skill
```

## Code Generation

```bash
# Watch mode (recommended during development)
dart run build_runner watch -d

# One-time build
dart run build_runner build -d

# Clean build (resolve conflicts)
dart run build_runner clean && dart run build_runner build -d
```

## Contributing

Contributions are welcome. Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/add-pattern`)
3. Follow existing documentation style
4. Submit a pull request

### Guidelines

- Keep `SKILL.md` under 500 lines
- Add detailed patterns to `references/` files
- Include working code examples
- Test with Riverpod 3.x and Freezed 3.x
- Follow the feature-first MVVM architecture guidance

## License

MIT - see [LICENSE](LICENSE)

## Resources

- [Riverpod Documentation](https://riverpod.dev)
- [Freezed Package](https://pub.dev/packages/freezed)
- [GoRouter Documentation](https://pub.dev/packages/go_router)
- [Drift Documentation](https://drift.simonbinder.eu/)
- [Flutter Documentation](https://flutter.dev/docs)
