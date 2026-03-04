# Open Authorization Protocol – DTU Logic Assignment

This repository contains the implementation and report for the DTU assignment on **protocol design and formal verification using OFMC**.

The goal of the assignment is to design and analyze an **open authorization protocol** where a user delegates access to a resource stored on a server to a third-party service using an **identity provider**.

The protocol is modeled in **AnB** and analyzed using **OFMC** under the **Dolev–Yao attacker model**.

---

# Group Members

- Kristófer Birgir Hjörleifsson — s253155  
- Mikael Máni Eyfeld Clarke — s253024  

DTU — Spring 2026

---

# Repository Structure
PROJECT1-LOGIC
│
├── anb
│ ├── week2_v1.anb
│ ├── week3_v1.anb
│ ├── week4_v1.anb
│ └── final_protocol.anb
│
├── logbook
│ └── logbook.md
│
├── report
│ └── report.tex
│
├── .gitignore
└── README.md

### `anb/`
Contains the protocol specifications written in **AnB** that are analyzed using OFMC.  
Each version corresponds to a weekly development step.

### `logbook/`
Contains the **development logbook**, documenting the evolution of the protocol, design decisions, and discovered attacks.

### `report/`
Contains the **LaTeX report** submitted for the assignment.

---

# Running OFMC

To verify a protocol version with OFMC:
ofmc anb/week3_v1.anb

OFMC will attempt to find attacks against the protocol under the Dolev–Yao attacker model.

---

# Building the Report

Compile the report using LaTeX:
pdflatex report/report.tex

You may need to run the command twice to resolve references.

---

# Assignment Overview

The protocol is developed incrementally during the course:

| Week | Topic |
|-----|------|
| Week 2 | Initial protocol + Dolev–Yao analysis |
| Week 3 | Identity provider + lazy intruder |
| Week 4 | Typing and protocol formats |
| Week 5 | TLS-style channels |
| Week 6 | Privacy and guessable password |

The final protocol and analysis are described in the report.

---

# Tools

- **OFMC** (Open Source Fixedpoint Model Checker)
- **AnB protocol language**
- **LaTeX** for the report
- **Git + GitHub** for collaboration

---

# Submission

The final submission includes:

- The report (PDF)
- AnB protocol files
- The lab logbook
