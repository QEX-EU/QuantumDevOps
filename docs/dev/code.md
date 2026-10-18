# Code

Good coding practices are the habits and conventions that make software easier to read, maintain, and extend; they include writing clear, self‑documenting code, using consistent naming and formatting, structuring projects with modular, reusable components, and adding automated tests to catch regressions early.  By embracing version control, continuous integration, and thorough documentation, developers reduce bugs, accelerate collaboration, and ensure that their code remains reliable and adaptable as requirements evolve.

<!-- # eScience Center Python Template

The eScience Center Python template is a Cookiecutter‑based starter kit that lets researchers generate a fully‑featured, standards‑compliant Python package in seconds; it includes ready‑to‑use project scaffolding (license, CI configuration, testing framework, documentation with Sphinx, code‑formatting tools, and a pre‑filled `pyproject.toml`) so developers can focus on scientific code rather than boiler‑plate setup, and it follows the Netherlands eScience Center’s Software Development Guide to promote reproducibility and best practices from day one.

[Get Started](https://github.com/NLeSC/python-template) -->

# Best Practices for Quantum Software

Quantum programs are still software: everything above applies. Quantum hardware and simulators add constraints (probabilistic outputs, noise, limited qubits, no state copying or inspection on real devices) that call for a few extra programming habits. This section focuses on how to write the code itself.

## Structure Your Program

- **Separate concerns.** Keep problem definition, circuit construction, execution (backend), and post-processing in distinct functions or modules, so each can be tested and swapped independently.
- **Build circuits from small, parameterized functions.** Compose reusable subcircuits (e.g. an ansatz layer, a QFT block) rather than writing one monolithic circuit. Parameters make circuits reusable and easier to optimize.
- **Stay backend-agnostic where possible.** Hide vendor-specific SDK calls behind a thin interface, or use portable representations such as OpenQASM, so you can move between simulators and hardware providers.
- **Treat the hybrid loop as a first-class component.** In variational algorithms (VQE, QAOA), the classical optimizer, the cost function, and the quantum evaluation should be clearly separated and individually configurable.
- **Use classical computation for what it does best.** Do pre- and post-processing classically and keep the quantum part as small as possible.

The two examples below do the same thing in Qiskit: prepare an n-qubit GHZ state and compute the probability of getting all zeros or all ones. The first mixes everything in one script. The second splits it into small functions.

=== "Bad"

    ```python
    from qiskit import QuantumCircuit
    from qiskit.primitives import StatevectorSampler

    # Circuit, backend, and post-processing are tangled together,
    # and the size (3) and shot count (1000) are hard-coded.
    qc = QuantumCircuit(3, 3)
    qc.h(0)
    qc.cx(0, 1)
    qc.cx(1, 2)
    qc.measure([0, 1, 2], [0, 1, 2])

    counts = StatevectorSampler().run([qc], shots=1000).result()[0].data.c.get_counts()
    print(counts["000"] / 1000 + counts["111"] / 1000)  # KeyError if an outcome is never sampled
    ```

    Changing the number of qubits, the backend, or the success criterion means editing the middle of the script, and none of the pieces can be tested on its own.

=== "Good"

    ```python
    from qiskit import QuantumCircuit
    from qiskit.primitives import StatevectorSampler


    def ghz_circuit(n_qubits: int) -> QuantumCircuit:
        """Circuit construction: prepare (|0...0> + |1...1>)/sqrt(2) and measure all qubits."""
        qc = QuantumCircuit(n_qubits, n_qubits)
        qc.h(0)
        for i in range(n_qubits - 1):
            qc.cx(i, i + 1)
        qc.measure(range(n_qubits), range(n_qubits))
        return qc


    def run(circuit: QuantumCircuit, sampler, shots: int) -> dict[str, int]:
        """Execution: the sampler is passed in, so any backend can be used."""
        result = sampler.run([circuit], shots=shots).result()
        return result[0].data.c.get_counts()


    def ghz_probability(counts: dict[str, int], n_qubits: int) -> float:
        """Post-processing: pure classical code, easy to unit test."""
        shots = sum(counts.values())
        all_zeros, all_ones = "0" * n_qubits, "1" * n_qubits
        return (counts.get(all_zeros, 0) + counts.get(all_ones, 0)) / shots


    if __name__ == "__main__":
        n = 3
        counts = run(ghz_circuit(n), StatevectorSampler(), shots=1000)
        print(ghz_probability(counts, n))  # 1.0 on an ideal simulator
    ```

    Each function has one job. You can test `ghz_probability` with hand-written counts, check `ghz_circuit` on an exact simulator for several sizes, and swap the sampler for a noisy simulator or hardware without touching the rest.

## Write Clear Circuits

- **Name qubits and registers meaningfully,** and use labels or comments to mark the intent of each block.
- **Avoid magic numbers.** Give qubit indices, angles, and shot counts names or parameters.
- **Document the algorithm, not only the code.** State the problem, the circuit's intent, assumptions, and the expected result, with a reference to the underlying paper where relevant.
- **Draw the circuit.** Visualizing it is often the quickest way to catch a wrong gate or qubit index.
- **Use the library's own building blocks** (standard gates, library circuits, primitives) instead of reimplementing them, and avoid deprecated APIs.

The two examples below build the same circuit in Qiskit: encode the integer 5 in a 4-qubit register, apply a quantum Fourier transform (QFT) followed by its inverse, and measure. The result should be 5 again.

=== "Bad"

    ```python
    from qiskit import QuantumCircuit

    qc = QuantumCircuit(4, 4)
    qc.x(0)
    qc.x(2)
    for j in range(4):
        qc.h(j)
        for k in range(j + 1, 4):
            qc.cp(3.14159 / 2 ** (k - j), k, j)
    # ... inverse written out by hand again, with its own indices ...
    qc.measure([0, 1, 2, 3], [0, 1, 2, 3])
    ```

    The qubit indices and the value 5 are hidden in `x` gates, nothing says what each block is for, and `pi` is approximated. The QFT is reimplemented by hand and omits the final qubit swaps, which makes it easy to get wrong. The inverse would have to be written out separately and kept in sync.

=== "Good"

    ```python
    from qiskit import ClassicalRegister, QuantumCircuit, QuantumRegister
    from qiskit.circuit.library import QFTGate

    N_BITS = 4
    VALUE = 5  # integer to encode, 0 <= VALUE < 2**N_BITS


    def qft_roundtrip(value: int, n_bits: int) -> QuantumCircuit:
        """Encode `value` in binary, apply the QFT and then its inverse, and measure."""
        number = QuantumRegister(n_bits, "number")
        result = ClassicalRegister(n_bits, "result")
        qc = QuantumCircuit(number, result)

        # Encode: qubit i holds bit i of `value` (little-endian)
        for i in range(n_bits):
            if (value >> i) & 1:
                qc.x(number[i])
        qc.barrier(label="encode")

        # Library QFT instead of a hand-written one
        qft = QFTGate(n_bits)
        qc.append(qft, number)
        qc.barrier(label="QFT")
        qc.append(qft.inverse(), number)
        qc.barrier(label="QFT^-1")

        qc.measure(number, result)
        return qc


    qc = qft_roundtrip(VALUE, N_BITS)
    print(qc.draw("text"))  # check the circuit by eye before running it
    ```

    Named registers, named constants, comments and labelled barriers make the intent visible in the code and in the drawing. The library's `QFTGate` handles the details, and `.inverse()` guarantees the second block matches the first. `QFTGate` requires a recent Qiskit (2.1 or later); on older versions use `qiskit.circuit.library.QFT`.

## Test Your Code

- **Test classical logic with ordinary unit tests.** Most of your code (data preparation, cost functions, result parsing) is deterministic.
- **Test circuits on exact simulators first.** Check statevectors, unitaries, or expected probabilities against known analytical results for small cases.
- **Test statistically for sampled outputs.** Use fixed random seeds and assert within tolerances derived from shot noise (roughly 1/√shots), rather than asserting exact counts.
- **Use property-based and invariant checks.** For example: circuit followed by its inverse gives identity, probabilities sum to 1, symmetries are preserved.
- **Test under noise.** Run noisy simulations before using real hardware.

Each tab below shows a small `pytest` example for the matching bullet.

=== "Classical logic"

    ```python
    import pytest

    from ghz import ghz_probability  # the post-processing function from the example above


    def test_ghz_probability_ideal():
        counts = {"000": 480, "111": 520}
        assert ghz_probability(counts, 3) == pytest.approx(1.0)


    def test_ghz_probability_missing_outcome():
        # An outcome that was never sampled must not raise a KeyError
        assert ghz_probability({"000": 1000}, 3) == pytest.approx(1.0)


    def test_ghz_probability_with_errors():
        counts = {"000": 250, "111": 250, "010": 500}
        assert ghz_probability(counts, 3) == pytest.approx(0.5)
    ```

    No quantum library is needed: hand-written counts are enough to test the classical code.

=== "Exact simulator"

    ```python
    import numpy as np
    from qiskit import QuantumCircuit
    from qiskit.quantum_info import Statevector


    def test_bell_state_vector():
        qc = QuantumCircuit(2)
        qc.h(0)
        qc.cx(0, 1)

        expected = Statevector(np.array([1, 0, 0, 1]) / np.sqrt(2))
        assert Statevector(qc).equiv(expected)  # equal up to a global phase
    ```

    No measurements and no shots: the state vector is compared with the analytical result.

=== "Statistical"

    ```python
    import numpy as np
    from qiskit import QuantumCircuit
    from qiskit.primitives import StatevectorSampler


    def test_bell_counts_within_shot_noise():
        shots = 4000
        qc = QuantumCircuit(2)
        qc.h(0)
        qc.cx(0, 1)
        qc.measure_all()

        sampler = StatevectorSampler(seed=1234)  # fixed seed: the test is repeatable
        counts = sampler.run([qc], shots=shots).result()[0].data.meas.get_counts()

        assert set(counts) <= {"00", "11"}  # forbidden outcomes never appear
        p00 = counts.get("00", 0) / shots
        sigma = np.sqrt(0.5 * 0.5 / shots)  # standard deviation of the estimate
        assert abs(p00 - 0.5) < 5 * sigma
    ```

    The tolerance comes from the shot noise (about 1/√shots), so the test neither fails by chance nor passes with a wrong circuit.

=== "Invariants"

    ```python
    import numpy as np
    import pytest
    from qiskit import QuantumCircuit
    from qiskit.circuit.random import random_circuit
    from qiskit.quantum_info import Operator, Statevector


    @pytest.mark.parametrize("seed", range(5))
    def test_circuit_followed_by_inverse_is_identity(seed):
        qc = random_circuit(3, depth=4, seed=seed)
        roundtrip = qc.compose(qc.inverse())
        assert Operator(roundtrip).equiv(Operator(QuantumCircuit(3)))


    @pytest.mark.parametrize("seed", range(5))
    def test_probabilities_sum_to_one(seed):
        qc = random_circuit(3, depth=4, seed=seed)
        assert np.sum(Statevector(qc).probabilities()) == pytest.approx(1.0)
    ```

    The checks hold for any circuit, so they can be run on many random inputs without knowing the answer in advance.

=== "Noise"

    ```python
    from qiskit import QuantumCircuit, transpile
    from qiskit_aer import AerSimulator
    from qiskit_aer.noise import NoiseModel, depolarizing_error


    def test_bell_state_under_depolarizing_noise():
        shots = 4000
        qc = QuantumCircuit(2)
        qc.h(0)
        qc.cx(0, 1)
        qc.measure_all()

        noise = NoiseModel()
        noise.add_all_qubit_quantum_error(depolarizing_error(0.05, 2), ["cx"])
        sim = AerSimulator(noise_model=noise, seed_simulator=1234)

        counts = sim.run(transpile(qc, sim), shots=shots).result().get_counts()
        p_correct = (counts.get("00", 0) + counts.get("11", 0)) / shots

        # Noise lowers the success probability (expected: about 0.975) but must not destroy it
        assert 0.9 < p_correct < 1.0
    ```

    Requires `pip install qiskit-aer`. Noisy simulations are slower, so mark them (for example with `@pytest.mark.slow`) and run them less often than the fast tests.

## Program with the Hardware in Mind

- **Keep circuits shallow and small.** Minimize qubit count, depth, and especially two-qubit gates; resources on current devices are scarce.
- **Know the device.** Qubit connectivity, native gate set, and gate fidelities determine what runs well.
- **Inspect the transpiled circuit** (depth, two-qubit gate count) rather than treating compilation as a black box.
- **Choose shot counts consciously,** balancing statistical precision against cost and queue time.
- **Develop locally, scale up gradually.** Start on an exact simulator, then a noisy one, and only then submit to hardware.

Each tab below shows a small Qiskit example for the matching bullet. The examples use `GenericBackendV2`, a configurable fake device, so they run without an account. With a real device, the same calls work on a backend obtained from your provider.

=== "Shallow circuits"

    ```python
    from qiskit import QuantumCircuit


    def ghz_linear(n: int) -> QuantumCircuit:
        qc = QuantumCircuit(n)
        qc.h(0)
        for i in range(n - 1):
            qc.cx(i, i + 1)  # a chain of n-1 sequential gates
        return qc


    def ghz_log_depth(n: int) -> QuantumCircuit:
        qc = QuantumCircuit(n)
        qc.h(0)
        filled = 1  # qubits 0..filled-1 already hold a copy of the state
        while filled < n:
            for i in range(min(filled, n - filled)):
                qc.cx(i, filled + i)  # the number of copies doubles at each step
            filled *= 2
        return qc


    print(ghz_linear(8).depth())     # 8
    print(ghz_log_depth(8).depth())  # 4
    ```

    Both circuits prepare the same state with the same number of two-qubit gates, but the second is half as deep, so less noise accumulates. Always re-check the depth after transpiling: on a device with limited connectivity, the long-range gates may need extra swaps.

=== "Know the device"

    ```python
    from qiskit.providers.fake_provider import GenericBackendV2

    # A 5-qubit device with its qubits connected in a line
    backend = GenericBackendV2(
        num_qubits=5,
        coupling_map=[[0, 1], [1, 2], [2, 3], [3, 4]],
        seed=42,
    )

    print(backend.operation_names)          # native gates, e.g. ['cx', 'id', 'rz', 'sx', 'x', ...]
    print(list(backend.coupling_map))       # which qubit pairs can interact directly
    print(backend.target["cx"][(0, 1)])     # error rate and duration of cx on qubits 0 and 1
    ```

    Check these before designing the circuit: only the listed gates run natively, only connected qubits can interact directly, and the error rates differ between qubits. For real hardware, replace the fake backend with `service.backend("<name>")`; the `target` information is the same, but contains actual calibration data.

=== "Inspect the transpiled circuit"

    ```python
    from qiskit import QuantumCircuit
    from qiskit.providers.fake_provider import GenericBackendV2
    from qiskit.transpiler.preset_passmanagers import generate_preset_pass_manager

    backend = GenericBackendV2(
        num_qubits=5,
        coupling_map=[[0, 1], [1, 2], [2, 3], [3, 4]],
        seed=42,
    )

    qc = QuantumCircuit(5)
    qc.h(0)
    qc.cx(0, 4)  # qubits 0 and 4 are far apart on this device
    qc.cx(0, 2)
    qc.cx(2, 4)  # 0, 2 and 4 are all mutually connected: impossible on a line
    qc.measure_all()

    pm = generate_preset_pass_manager(optimization_level=2, backend=backend, seed_transpiler=1)
    isa = pm.run(qc)

    print("depth:          ", isa.depth())
    print("two-qubit gates:", isa.num_nonlocal_gates())
    print("gate counts:    ", dict(isa.count_ops()))
    print("original depth: ", qc.depth(), "| original two-qubit gates:", qc.num_nonlocal_gates())
    ```

    The original circuit has three two-qubit gates. After transpiling, there are four (with this seed) because the three qubits it connects cannot all be neighbors on a line, so the router had to insert a swap. Compare these numbers across optimization levels and qubit layouts, and treat a large increase as a sign that the circuit does not fit the device.

=== "Shot counts"

    ```python
    import math


    def shots_for_precision(eps: float, p: float = 0.5, n_sigma: float = 1.0) -> int:
        """Shots needed so that an estimated probability is within eps of the true value.

        The standard deviation of the estimate is sqrt(p(1-p)/shots). p=0.5 is the worst case.
        """
        return math.ceil(round(n_sigma**2 * p * (1 - p) / eps**2, 9))


    print(shots_for_precision(0.01))               # 2500   (+/- 0.01 at one standard deviation)
    print(shots_for_precision(0.01, n_sigma=3))    # 22500  (+/- 0.01 at three standard deviations)
    print(shots_for_precision(0.001))              # 250000 (10x better precision costs 100x the shots)
    ```

    Precision improves only with the square root of the number of shots, so decide how precise a result needs to be before choosing a number. Use few shots while developing and more only for final runs. On hardware, also consider the per-job shot limit and the cost of the run.

=== "Develop locally"

    ```python
    from qiskit import QuantumCircuit, transpile
    from qiskit.primitives import StatevectorSampler
    from qiskit.providers.fake_provider import GenericBackendV2
    from qiskit_aer import AerSimulator

    qc = QuantumCircuit(3)
    qc.h(0)
    qc.cx(0, 1)
    qc.cx(1, 2)
    qc.measure_all()
    shots = 1000

    # Stage 1: exact, noiseless simulation (fast, small circuits)
    counts = StatevectorSampler().run([qc], shots=shots).result()[0].data.meas.get_counts()
    print("ideal:", counts)

    # Stage 2: noisy simulation using the noise and layout of a (fake) device
    backend = GenericBackendV2(num_qubits=5, coupling_map=[[0, 1], [1, 2], [2, 3], [3, 4]], seed=42)
    noisy = AerSimulator.from_backend(backend)
    counts = noisy.run(transpile(qc, noisy), shots=shots).result().get_counts()
    print("noisy:", counts)

    # Stage 3: real hardware (needs an account; run only when stages 1 and 2 look right)
    # from qiskit_ibm_runtime import QiskitRuntimeService, SamplerV2
    # service = QiskitRuntimeService()
    # device = service.least_busy(operational=True, simulator=False)
    # isa = generate_preset_pass_manager(backend=device, optimization_level=2).run(qc)
    # job = SamplerV2(mode=device).run([isa], shots=shots)
    ```

    Each stage costs more time and money than the previous one, so only move on once the current stage gives the expected results. The ideal run should show only `000` and `111`; the noisy run should show a few other outcomes; if it shows many, the circuit is probably too deep for the device.

## Keep Results Reproducible

- **Fix random seeds** for simulators, transpilation, and optimizers.
- **Keep logic in importable modules,** with notebooks used only for exploration and plotting.
- **Record the essentials of each run** in your results: SDK version, backend, number of shots, and any error mitigation applied.

Each tab below shows a small example for the matching bullet.

=== "Fix random seeds"

    ```python
    import numpy as np
    from qiskit import QuantumCircuit, transpile
    from qiskit.primitives import StatevectorSampler
    from qiskit.providers.fake_provider import GenericBackendV2
    from qiskit_aer import AerSimulator

    SEED = 1234
    SHOTS = 1000

    qc = QuantumCircuit(3)
    qc.h(0)
    qc.cx(0, 1)
    qc.cx(1, 2)
    qc.measure_all()

    backend = GenericBackendV2(
        num_qubits=5, coupling_map=[[0, 1], [1, 2], [2, 3], [3, 4]], seed=SEED
    )

    # 1. Transpilation: the layout and routing passes are randomized
    isa = transpile(qc, backend, optimization_level=3, seed_transpiler=SEED)

    # 2. Sampling on an ideal simulator
    ideal = StatevectorSampler(seed=SEED).run([qc], shots=SHOTS).result()[0].data.meas.get_counts()

    # 3. Sampling on a noisy simulator
    noisy_sim = AerSimulator.from_backend(backend)
    noisy = noisy_sim.run(isa, shots=SHOTS, seed_simulator=SEED).result().get_counts()

    # 4. Classical optimizers and initial parameters
    rng = np.random.default_rng(SEED)
    initial_point = rng.uniform(0, 2 * np.pi, size=3)
    ```

    Running this twice gives identical transpiled circuits, counts, and initial parameters. Define the seed once, pass it everywhere, and store it with the results. A fixed seed makes a simulator run repeatable, but not a hardware run, which is why the record in the third tab matters.

=== "Importable modules"

    ```text
    my_project/
    ├── pyproject.toml
    ├── src/my_project/
    │   ├── circuits.py      # ghz_circuit(), ansatz(), ...
    │   ├── execution.py     # run(), backend selection
    │   └── analysis.py      # ghz_probability(), plotting helpers
    ├── tests/
    │   └── test_circuits.py
    └── notebooks/
        └── explore_ghz.ipynb
    ```

    ```python
    # notebooks/explore_ghz.ipynb: the notebook only calls the package and plots
    from qiskit.primitives import StatevectorSampler

    from my_project.analysis import ghz_probability
    from my_project.circuits import ghz_circuit
    from my_project.execution import run

    n = 4
    counts = run(ghz_circuit(n), StatevectorSampler(seed=1234), shots=1000)
    print(ghz_probability(counts, n))
    ```

    Code in a notebook cannot be imported, tested, or reviewed easily, and it depends on the order in which the cells were run. Code in modules can be tested with `pytest` (see "Test Your Code") and reused by scripts and other notebooks. Install the package in editable mode with `pip install -e .` so that notebooks always import the current version.

=== "Record each run"

    ```python
    import json
    import platform
    from datetime import datetime, timezone
    from importlib.metadata import version
    from pathlib import Path


    def save_run(path, counts, *, backend, shots, seed, optimization_level, mitigation="none"):
        """Save the results together with everything needed to repeat them."""
        record = {
            "timestamp": datetime.now(timezone.utc).isoformat(timespec="seconds"),
            "python": platform.python_version(),
            "qiskit": version("qiskit"),
            "qiskit-aer": version("qiskit-aer"),
            "backend": backend.name,
            "shots": shots,
            "seed": seed,
            "optimization_level": optimization_level,
            "error_mitigation": mitigation,
            "counts": counts,  # raw results, before any post-processing
        }
        Path(path).write_text(json.dumps(record, indent=2))


    save_run(
        "results/ghz_noisy.json", noisy,
        backend=backend, shots=SHOTS, seed=SEED, optimization_level=3,
    )
    ```

    This example continues the first tab. The file records the library versions, backend, shots, seed, transpiler settings, and mitigation next to the raw counts. For hardware runs, also store the job ID and the date, since the calibration data changes between runs. Never store credentials in this record.

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

=== "Qiskit"

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

=== "Cirq"

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

=== "PennyLane"

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

=== "Amazon Braket"

    ```python
    from braket.circuits import Circuit
    from braket.devices import LocalSimulator

    circuit = Circuit().h(0).cnot(0, 1)

    result = LocalSimulator().run(circuit, shots=1000).result()
    print(result.measurement_counts)  # e.g. Counter({'00': 505, '11': 495})
    ```

    Replace `LocalSimulator()` with `AwsDevice(<device ARN>)` to run on a managed simulator or on quantum hardware (requires an AWS account).

=== "CUDA-Q"

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

=== "Qibo"

    ```python
    from qibo import Circuit, gates

    c = Circuit(2)
    c.add(gates.H(0))
    c.add(gates.CNOT(0, 1))
    c.add(gates.M(0, 1))

    result = c(nshots=1000)
    print(result.frequencies())  # e.g. Counter({'00': 503, '11': 497})
    ```

=== "Qrisp"

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

## Running a Qiskit LLM Locally with Ollama

[Ollama](https://ollama.com) runs open models on your own machine, so prompts and code never leave it. It can pull GGUF-format models directly from Hugging Face, which makes it a simple way to try the Qiskit Granite models. A GPU or an Apple Silicon Mac is recommended; an 8B model at 4-bit quantization needs roughly 6–8 GB of free memory.

1. **Install Ollama.** Download it from [ollama.com/download](https://ollama.com/download), or on macOS use `brew install ollama`. On Linux:

   ```bash
   curl -fsSL https://ollama.com/install.sh | sh
   ```

2. **Start the server** (the desktop app does this automatically):

   ```bash
   ollama serve
   ```

3. **Find the model.** Browse the [`Qiskit` organization on Hugging Face](https://huggingface.co/Qiskit) and pick a Granite Qiskit model that is published in GGUF format. If only full-precision weights are published, use a GGUF conversion from the model card or convert it yourself with `llama.cpp`.

4. **Pull and run it.** Ollama accepts the `hf.co/<organization>/<repository>` syntax, with an optional quantization tag:

   ```bash
   ollama run hf.co/Qiskit/<model-repository>:Q4_K_M
   ```

   Replace `<model-repository>` with the repository name from step 3. The first run downloads the model; later runs start immediately.

5. **Test it.** Ask for something with a known answer and check it against the library version you pin:

   ```text
   >>> Write a Qiskit circuit that prepares a Bell state and samples it 1000 times using StatevectorSampler.
   ```

6. **Use it from your editor or code.** Ollama serves an HTTP API on `http://localhost:11434`, including an OpenAI-compatible endpoint at `/v1`. Point any compatible client at it, for example the Continue extension for VS Code or JupyterLab, or call it directly:

   ```bash
   curl http://localhost:11434/api/generate -d '{
     "model": "hf.co/Qiskit/<model-repository>:Q4_K_M",
     "prompt": "Write a Qiskit circuit that prepares a GHZ state on 3 qubits.",
     "stream": false
   }'
   ```

   The Qiskit Code Assistant VS Code extension also documents how to use a local Ollama model instead of the hosted service; see its README for the current settings.

Tips:

- **Set a system prompt** with your pinned Qiskit version (for example "Use Qiskit 2.x, primitives V2, no `execute()`") through a `Modelfile` and `ollama create`, so every session starts with the right context.
- **Pick a quantization that fits.** `Q4_K_M` is a good default; use `Q8_0` if you have the memory and want better quality.
- **Keep it on localhost.** Ollama has no authentication; do not expose port 11434 to a network.
- **Verify the output** exactly as with a hosted assistant. A smaller local model is not more reliable.

Model repository names and quantization tags change, so check the model card on Hugging Face for the current ones.

## Recommendations

- **Prefer a Qiskit-specialized LLM for Qiskit code.** It is less likely to suggest deprecated APIs and is better at idiomatic Qiskit than a general model. Use a general assistant for the surrounding classical code, tests, and documentation.
- **Give the assistant context.** State the pinned Qiskit version, the target backend, and any constraints (qubit count, connectivity, native gates) in the prompt or in a project instruction file.
- **Verify every output.** Run generated circuits on an exact simulator, compare against analytical results, and inspect the transpiled circuit. Passing code is not the same as correct physics.
- **Never paste credentials.** Keep IBM Quantum API tokens and other secrets out of prompts, notebooks shared with assistants, and committed files.
- **Record AI use in your reproducibility notes.** Note which assistant and model produced or modified significant parts of the code, along with the human review that followed.
- **Check licensing and data policies.** Know whether your assistant sends code to an external service, and whether that is acceptable for your project.
