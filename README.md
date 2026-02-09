# UMB: A Unified Markov Binary Format

The unified Markov binary (UMB) format is an efficient, extensible and well-supported
file format for explicit-state representation of probabiistic models.
It is designed to facilitate the exchange of a wide range of stochastic models, including:

* discrete-time Markov chains (DTMCs)
* continuous-time Markov chains (CTMCs)
* Markov decision processes (MDPs)
* Markov automata
* partially observable MDPs (POMDPs)
* interval MDPs and DTMCs
* stochastic games

Model import and export is already supported by popular probabilistic model checkers:

* [PRISM](https://www.prismmodelchecker.org/)
* [Storm](https://www.stormchecker.org/)
* [Modest](https://www.modestchecker.net/)

There is dedicated tool support in the form of:

* [UMBI](https://github.com/pmc-tools/umbi/): a lightweight Python library and
  command-line interface for reading, manipulating, creating, and validating UMB files.

* The [UMB observatory](https://github.com/pmc-tools/umb-observatory/): a shared test environment
  for continuous integration testing of UMB-supporting software. 

For more details, see the formal [specification](spec.md) or a simple [example](example/README.md).
