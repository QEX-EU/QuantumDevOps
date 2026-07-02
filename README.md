# Quantum Dev Ops Guide Book

To contribute to the book

1. Clone the repo: `git clone https://github.com/QEX-EU/QuantumDevOps`
2. Change directory `cd QuantumDevOps`
3. Install dependencies: `uv pip install mkdocs mkdocs-jupyter mkdocs-literate-nav"`
4. Launch mkdocs: `mkdocs serve`

This will launch the guide in a webbrowser. You can edit/add  markdown files and it will autorefreseh. 
Once you are happy with your changes commit upload them: `mkdocs gh-deploy`
Don't forget to commit your changes to the repo as well.

# QEX - Task 4.5: QuantumDevOps

Here we collect ideas for tools to be used in the QuantumDevOps workflow, so to discuss them during TEM.

So far we have:

- Gluing component: [EasyBuild](https://easybuild.io)
- PLAN - Requirements + Design:
  - ...
- CODE - Implementation:
  - [Tool to setup a new Python package](https://github.com/NLeSC/python-template.git), which could be adapted as the first task of the Dev step.
- BUILD - Transpile:
  - ...
- TEST - Simulate + Simulate Noise + Run on QC backend:
  - ...
- RELEASE - Package + Version + Chagelog:
  - ...
- EVALUATE - Evaluate QC Backend + Select QC Platform:
  - ...
- DEPLOY/CONFIGURE - Transpile + Configure + Deploy:
  - ...
- MONITOR - Monitoring Platform + Monitoring Execution + Result Collection:
  - ...
- FEEDBACK - Execution Time + Optimization Rate + Result Distrbution:
  - ...

## General Remarks

The tools we identify should be able to be or become coding language agnostic.
