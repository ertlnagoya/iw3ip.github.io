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
6. [Quickstart](workshop/quickstart.md)
7. [Hands-on](hands-on/index.md)
8. [References](foundations/references.md)

## For Students Interested in the Laboratory

If you would like to work on this area in the laboratory, please apply through the following affiliation.

- Undergraduate students:
  - Information Systems area, Department of Computer Science, School of Informatics
- Graduate students:
  - Department of Information Systems, Graduate School of Informatics
  - Takada-Matsubara Laboratory

<span style="color: #c62828; font-weight: 700;">Research students are generally not accepted.</span>

If you aim to obtain a doctoral degree, please consult us about entering the doctoral program.

For details on graduate school admissions, see:

- Japanese: <https://www.i.nagoya-u.ac.jp/gs/entranceexamination/>
- English: <https://www.i.nagoya-u.ac.jp/en/gs/entranceexamination/>

## Run docs locally

```bash
pip install mkdocs-material mkdocs-static-i18n
mkdocs serve
```

Open: `http://127.0.0.1:8000`
