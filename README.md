# jasmine
A digital brain. Built with love by @levermake, in collaboration with Codex.

Jasmine is designed as an open-source, browser-accessible cognitive system with persistent memory, evidence-grounded reasoning, controlled tools, and evaluated learning.

## Project status

Architecture and implementation planning. The documents below specify the system to build; this repository does not yet contain a working cognitive application.

## Build specification

- [Architecture](docs/architecture.md): cognitive subsystems, memory schema, data flows, APIs, execution boundaries, scaling, and hard problems.
- [Technology register](docs/technology-register.md): component choices, upstream sources, licenses, model compatibility, hardware, and deployment constraints.
- [Implementation roadmap](docs/implementation-roadmap.md): build increments, acceptance criteria, evaluation, and the path to bounded self-directed learning.

Start with a modular Python application and worker, PostgreSQL with pgvector, a local language model, and a React browser client. Add distributed services and model training when measured requirements justify them.

Jasmine's design targets measurable capabilities such as reliable recall, source-backed answers, constrained problem solving, and reusable verified workflows. General human-level cognition or unbounded self-improvement is a research aspiration, not a promised feature.

## License

Jasmine uses the existing [MIT license](LICENSE). Dependencies, model weights, and datasets retain their respective licenses.
