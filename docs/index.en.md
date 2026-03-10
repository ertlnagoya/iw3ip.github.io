# IoTxWeb3 Intelligence Platform (IW3IP) Documentation

This site is organized so that readers can move step by step through the IW3IP overview, implementation examples, and hands-on exercises.

## Main Use Cases

- Undergraduate students studying computing or information systems
- Instructors and TAs running practical sessions

[Learning Foundations / Course Guide](foundations/course-guide.md) is the best starting point for understanding the overall structure.

## What This Site Covers

This site is organized so that readers can work through the core concepts and the introductory hands-on path within the documentation itself.

- Foundations:
  - blockchain
  - Hardhat
  - SSI / DID / VC
- Practice:
  - quickstart
  - hands-on samples
  - troubleshooting

External websites and papers are positioned as follow-up material for standards, deeper technical detail, and research background.

## Documentation model

- **Workshop**: instructor/TA flow design (goals, timing, orchestration)
- **Hands-on**: participant execution steps (commands, expected results)

In short, **Hands-on is a part of Workshop**.

## Start here

1. [Workshop / Prerequisites](workshop/prerequisites.md)
2. [Learning Foundations / Learning Roadmap](foundations/roadmap.md)
3. [Workshop / Quickstart](workshop/quickstart.md)
4. [Hands-on](hands-on/index.md)
5. [Operations / Troubleshooting](operations/troubleshooting.md)

## If You Want To Run One Demo First

If you want the simplest Phase 1 / Phase 2 entry without physical devices, `ha-demo-simulator` is the best starting point.

- Matching page: [Home Assistant Demo Simulator sample](hands-on/ha-demo-simulator.md)
- What you can confirm:
  - `Home Assistant demo -> MQTT -> publisher`
  - state sharing for `temperature` and `power`
  - event sharing for `flood_risk_high` and `possible_littering`
  - `allowed`, `denied`, and the `audit log`

Start command:

```bash
docker compose -f infra/docker-compose.yml --profile ha-demo up --build -d
```

Open:

- `http://localhost:8123`
- `http://localhost:8080/health`

After that, `assistant-demo` is the shortest one-command path into Phase 3 and intelligence integration.

As the shortest Phase 3 demo path, `assistant-demo` starts the following in one command:

- `assistant-demo`
- `llm-mock`
- `assistant-ui`

Matching page:

- [LLM Planner Hands-on](hands-on/llm-planner.md)

Start command:

```bash
docker compose -f infra/docker-compose.yml --profile assistant-demo up --build -d
```

Open:

- `http://localhost:4173`

## Recommended Learning Order

1. [Course Guide](foundations/course-guide.md)
2. [Learning Roadmap](foundations/roadmap.md)
3. [Blockchain Basics](foundations/blockchain-basics.md)
4. [Hardhat Basics](foundations/hardhat-basics.md)
5. [SSI/DID/VC Basics](foundations/ssi-did-vc-basics.md)
6. [Home Assistant Demo Simulator sample](hands-on/ha-demo-simulator.md)
7. [Quickstart](workshop/quickstart.md)
8. [Hands-on](hands-on/index.md)
9. [References](foundations/references.md)

## Run docs locally

```bash
pip install mkdocs-material mkdocs-static-i18n
mkdocs serve
```

Open: `http://127.0.0.1:8000`
