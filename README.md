# Statamic with Eloquent Driver Skill

A [Laravel Skill](https://skills.laravel.com) for AI-assisted development with [Statamic CMS](https://statamic.com) and the [Eloquent Driver](https://github.com/statamic/eloquent-driver).

This skill teaches your AI coding assistant how to work with Statamic's Blade-only templating, the Eloquent Driver's hybrid storage strategy, content querying via facades, Value object handling, navigation rendering, blueprints, and more.

## Installation

### Via Laravel Boost (recommended)

```bash
php artisan boost:add-skill statamic-with-eloquent
```

### Manual installation

1. Create the directory `skills/statamic-with-eloquent/` in your project root
2. Copy `skills/statamic-with-eloquent/SKILL.md` into that directory
3. Your AI assistant will automatically pick up the skill when working in the project

## What's included

- Blade-only templating guidelines (no Antlers)
- Eloquent Driver hybrid storage configuration
- Content querying via Statamic facades
- Value object handling in templates and PHP
- Blueprint conventions and common fieldtypes
- Navigation rendering with `<statamic:nav>` tags
- View Composer patterns
- Asset handling
- PHPStan considerations for Statamic's dynamic properties
- Common pitfalls and how to avoid them

## License

MIT — see [LICENSE](LICENSE) for details.
