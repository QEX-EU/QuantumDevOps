# Code

Good coding practices are the habits and conventions that make software easier to read, maintain, and extend; they include writing clear, self‑documenting code, using consistent naming and formatting, structuring projects with modular, reusable components, and adding automated tests to catch regressions early.  By embracing version control, continuous integration, and thorough documentation, developers reduce bugs, accelerate collaboration, and ensure that their code remains reliable and adaptable as requirements evolve.

# eScience Center Python Template

The eScience Center Python template is a Cookiecutter‑based starter kit that lets researchers generate a fully‑featured, standards‑compliant Python package in seconds; it includes ready‑to‑use project scaffolding (license, CI configuration, testing framework, documentation with Sphinx, code‑formatting tools, and a pre‑filled `pyproject.toml`) so developers can focus on scientific code rather than boiler‑plate setup, and it follows the Netherlands eScience Center’s Software Development Guide to promote reproducibility and best practices from day one.

[Get Started](https://github.com/NLeSC/python-template)

# Best Practices for Quantum Software

Quantum programs are still software: everything above applies. Quantum hardware and simulators add constraints (probabilistic outputs, noise, limited qubits, no state copying or inspection on real devices) that call for a few extra habits.

## Design

- **Separate concerns.** Keep problem definition, circuit construction, execution (backend), and post-processing in distinct modules, so each can be tested and swapped independently.
- **Build circuits from small, parameterized functions.** Compose reusable subcircuits (e.g. an ansatz layer, a QFT block) rather than writing one monolithic circuit. Parameters make circuits reusable and easier to optimize.
- **Stay backend-agnostic where possible.** Hide vendor-specific SDK calls behind a thin interface, or use portable representations such as OpenQASM, so you can move between simulators and hardware providers.
- **Use classical computation for what it does best.** Do pre- and post-processing classically, keep circuits shallow, and minimize qubit count and depth; resources on current devices are scarce.
- **Treat the hybrid loop as a first-class component.** In variational algorithms (VQE, QAOA), the classical optimizer, the cost function, and the quantum evaluation should be clearly separated and individually configurable.

## Testing

- **Test classical logic with ordinary unit tests.** Most of your code (data preparation, cost functions, result parsing) is deterministic.
- **Test circuits on exact simulators first.** Check statevectors, unitaries, or expected probabilities against known analytical results for small cases.
- **Test statistically for sampled outputs.** Use fixed random seeds and assert within tolerances derived from shot noise (roughly 1/√shots), rather than asserting exact counts.
- **Use property-based and invariant checks.** For example: circuit followed by its inverse gives identity, probabilities sum to 1, symmetries are preserved.
- **Test under noise.** Run noisy simulations with realistic noise models before using real hardware, and keep a small set of hardware tests separate from the main CI suite.

## Reproducibility

- **Record everything needed to rerun an experiment:** SDK and transpiler versions, backend name and calibration data or noise model, number of shots, random seeds, transpiler optimization level, and qubit layout.
- **Pin dependencies.** Quantum SDKs change quickly and APIs and transpilation results differ between releases.
- **Version-control circuits and parameters,** and store raw results (counts, measurement data) alongside processed results.
- **Use workflow tooling** (notebooks for exploration, scripts or pipelines for production runs) and make notebooks reproducible by keeping logic in importable modules.

## Performance and Hardware Awareness

- **Know the device.** Check qubit connectivity, native gate set, gate fidelities, and coherence times; design circuits to fit them.
- **Transpile deliberately.** Inspect the transpiled circuit (depth, two-qubit gate count) rather than treating compilation as a black box.
- **Apply error mitigation where useful** (readout correction, zero-noise extrapolation, dynamical decoupling), and document what was applied, since it affects how results should be interpreted.
- **Choose shot counts consciously,** balancing statistical precision against cost and queue time.
- **Develop locally, scale up gradually.** Use a simulator (statevector, then noisy, then tensor-network or similar for larger cases) before submitting to hardware or HPC-based simulators; budget for queue time and costs.

## Maintainability and Collaboration

- **Document the algorithm, not only the code.** State the problem, the circuit's intent, assumptions, and the expected result, with references to the underlying paper where relevant.
- **Name qubits and registers meaningfully,** and annotate circuits with comments or barriers/labels where they aid readability and visualization.
- **Visualize and review circuits.** Drawing a circuit is often the quickest way to catch mistakes.
- **Automate with CI.** Run linting, type checks, and simulator-based tests on every change, and cache or limit expensive runs.
- **Mind licensing and provenance** of algorithms, datasets, and any third-party quantum libraries you depend on.

# Major Quantum Computing Libraries

Most libraries are Python packages for building circuits, simulating them, and running them on hardware. The examples below all prepare a two-qubit Bell state, (|00⟩ + |11⟩)/√2, and sample it 1000 times. Ideal results show only `00` and `11`, each roughly half the time. Library APIs change between releases, so check the documentation for the version you pin.

| Library | Maintainer | Focus | Install |
|---|---|---|---|
| [Qiskit](https://qiskit.org) | IBM | General circuit programming, transpilation, IBM hardware | `pip install qiskit` |
| [Cirq](https://quantumai.google/cirq) | Google | NISQ circuits with explicit hardware-aware qubit layouts | `pip install cirq` |
| [PennyLane](https://pennylane.ai) | Xanadu | Quantum machine learning, differentiable and hybrid programming | `pip install pennylane` |
| [Amazon Braket SDK](https://github.com/amazon-braket/amazon-braket-sdk-python) | AWS | Access to multiple hardware vendors and simulators via AWS | `pip install amazon-braket-sdk` |
| [CUDA-Q](https://nvidia.github.io/cuda-quantum/) | NVIDIA | GPU-accelerated simulation, hybrid quantum-classical HPC workflows | `pip install cudaq` |
| [Qibo](https://qibo.science) | Qibo community | Simulation, hardware control, and optimization | `pip install qibo` |
| [Qrisp](https://qrisp.eu) | Fraunhofer FOKUS | High-level programming with quantum variables, automatic uncomputation, and compilation | `pip install qrisp` |

## Qiskit

```python
from qiskit import QuantumCircuit
from qiskit.primitives import StatevectorSampler

qc = QuantumCircuit(2, 2)
qc.h(0)
qc.cx(0, 1)
qc.measure([0, 1], [0, 1])

result = StatevectorSampler().run([qc], shots=1000).result()
print(result[0].data.c.get_counts())  # e.g. {'00': 507, '11': 493}
```

## Cirq

```python
import cirq

q0, q1 = cirq.LineQubit.range(2)
circuit = cirq.Circuit(
    cirq.H(q0),
    cirq.CNOT(q0, q1),
    cirq.measure(q0, q1, key="m"),
)

result = cirq.Simulator().run(circuit, repetitions=1000)
print(result.histogram(key="m"))  # Counter({0: ~500, 3: ~500})
```

## PennyLane

```python
import pennylane as qml

dev = qml.device("default.qubit", wires=2, shots=1000)

@qml.qnode(dev)
def bell():
    qml.Hadamard(wires=0)
    qml.CNOT(wires=[0, 1])
    return qml.counts()

print(bell())  # e.g. {'00': 498, '11': 502}
```

## Amazon Braket

```python
from braket.circuits import Circuit
from braket.devices import LocalSimulator

circuit = Circuit().h(0).cnot(0, 1)

result = LocalSimulator().run(circuit, shots=1000).result()
print(result.measurement_counts)  # e.g. Counter({'00': 505, '11': 495})
```

Replace `LocalSimulator()` with `AwsDevice(<device ARN>)` to run on a managed simulator or on quantum hardware (requires an AWS account).

## CUDA-Q

```python
import cudaq

@cudaq.kernel
def bell():
    q = cudaq.qvector(2)
    h(q[0])
    x.ctrl(q[0], q[1])
    mz(q)

print(cudaq.sample(bell, shots_count=1000))  # { 00:~500 11:~500 }
```

## Qibo

```python
from qibo import Circuit, gates

c = Circuit(2)
c.add(gates.H(0))
c.add(gates.CNOT(0, 1))
c.add(gates.M(0, 1))

result = c(nshots=1000)
print(result.frequencies())  # e.g. Counter({'00': 503, '11': 497})
```

## Qrisp

```python
from qrisp import QuantumVariable, h, cx

qv = QuantumVariable(2)
h(qv[0])
cx(qv[0], qv[1])

print(qv.get_measurement(shots=1000))  # e.g. {'00': 0.51, '11': 0.49}
```

Qrisp works at a higher level of abstraction than gate-by-gate circuits: `QuantumVariable`s can represent integers, floats, or custom types, and the compiler handles qubit allocation and automatic uncomputation of temporary values. Note that results are returned as probabilities rather than raw counts.

## Choosing a Library

- **Targeting a specific hardware provider:** use that provider's native stack (Qiskit for IBM, Braket for AWS-hosted devices), or a plugin that bridges to it.
- **Variational algorithms and quantum machine learning:** PennyLane offers automatic differentiation and integrates with PyTorch, JAX, and TensorFlow.
- **Large-scale simulation on GPUs or HPC systems:** CUDA-Q (and GPU-enabled backends in other libraries) are designed for this.
- **High-level, structured quantum programming:** Qrisp lets you write algorithms in terms of quantum variables and functions, leaving allocation and uncomputation to the compiler.
- **Portability:** keep library-specific code behind a thin interface, and consider exchanging circuits as OpenQASM so you can switch libraries later.

# AI Coding Assistants for Quantum Computing

General-purpose AI coding assistants (GitHub Copilot, Claude Code, ChatGPT, and similar) can speed up quantum software development, but they are weaker here than for mainstream languages. Quantum SDKs are small niches in the training data, their APIs change quickly, and models often produce code that looks plausible but uses removed or renamed functions (for example the pre-1.0 `execute()` and `Aer` workflows or the old `qiskit.providers.ibmq` package in Qiskit). They can also write circuits that run without errors but compute the wrong thing. Treat generated code as a draft: review it, run it on a simulator, and check it against the testing practices above.

## Qiskit Code Assistant and Qiskit LLMs

If you work with Qiskit, use an assistant that has been specialized for it rather than relying only on a general model. IBM and the Qiskit community provide the following:

- **Qiskit Code Assistant.** A code-completion and chat assistant based on IBM Granite models fine-tuned on Qiskit code and documentation. It integrates with VS Code and JupyterLab and is aware of current Qiskit idioms, such as primitives, transpiler passes, and the `qiskit-ibm-runtime` workflow.
- **Open Qiskit models.** The fine-tuned Granite Qiskit models are published on Hugging Face (under the `Qiskit` organization), so you can run them locally or on your own infrastructure. This matters when code or research must not leave your environment.
- **Qiskit HumanEval.** A benchmark of Qiskit coding tasks that is used to measure how well a model writes correct Qiskit code. Use it, or a small test set of your own, to compare assistants rather than relying on impressions.
- **Qiskit MCP servers.** The Model Context Protocol lets general agents such as Claude Code call Qiskit tooling (for example the Code Assistant, transpiler, or IBM Quantum backends) as tools, combining a general-purpose agent with Qiskit-specific knowledge.

Availability, model names, and access requirements change often, so check the [Qiskit documentation](https://docs.quantum.ibm.com) for the current setup.

## Recommendations

- **Prefer a Qiskit-specialized LLM for Qiskit code.** It is less likely to suggest deprecated APIs and is better at idiomatic Qiskit than a general model. Use a general assistant for the surrounding classical code, tests, and documentation.
- **Give the assistant context.** State the pinned Qiskit version, the target backend, and any constraints (qubit count, connectivity, native gates) in the prompt or in a project instruction file.
- **Verify every output.** Run generated circuits on an exact simulator, compare against analytical results, and inspect the transpiled circuit. Passing code is not the same as correct physics.
- **Never paste credentials.** Keep IBM Quantum API tokens and other secrets out of prompts, notebooks shared with assistants, and committed files.
- **Record AI use in your reproducibility notes.** Note which assistant and model produced or modified significant parts of the code, along with the human review that followed.
- **Check licensing and data policies.** Know whether your assistant sends code to an external service, and whether that is acceptable for your project.
