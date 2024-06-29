# CEA

!!! note ""
    Visit the [official website](https://www.cea.fr/) (French) or [Wikipedia](https://en.wikipedia.org/wiki/French_Alternative_Energies_and_Atomic_Energy_Commission).

The CEA (Commissariat à l'Énergie Atomique et aux Énergies Alternatives) is a French public government-funded research organisation in the areas of energy, defense and security, information technologies and health technologies.

The CEA has around 21 000 staff and owns multiple of the largest supercomputers ever built, which are used to simulate the weather, natural catastrophes or nuclear reactions. Since these topics may be of national interest, the CEA also has one of the best defensive cyber-security teams in France.

## Miasm

!!! note ""
    Visit the [official website](https://miasm.re/blog/).

Miasm is an open source reverse engineering framework in Python developed by the CEA's cyber-security team. It can disassemble binaries, emulate them in a sandbox, edit them live and otherwise analyse their behavior.

Miasm is used by experts to understand potential attacks or attempts at data exfiltration.

However, the implementation being in Python makes the tool slow to use, which is becoming more and more of a problem as malicious binaries get larger and larger. The team has started rewriting parts of Miasm to Rust, creating Python bindings to attempt minimal migration efforts for existing users.

## My involvement

As a Software Engineer intern during my studies at [ENSEIRB-MATMECA](enseirb.md), I was tasked with helping with the Python to Rust migration, as well as optimizing the Rust code. 

I brought performance improvements of up to 97% time reduction for some algorithms, and helped structure the codebase to ease maintenance. I helped to implement algorithms from active research, and created a custom expression single-pass simplification engine inspired by RegExp engines that was 30% faster than the previous one.

I also contributed some quality-of-life improvements to our open source dependencies, including a [prototype Python type exporter for PyO₃](https://github.com/PyO3/pyo3/issues/2454) to convert Rust type information to Python.

## Technical stack

- Python
- Rust
