##  GPTX: Comparing multi-processing features of Rust and CPP using GPT-2

[[ View on Github ]](https://github.com/arunpatro/gptx)

In this project, we benchmark the multiprocessing features of Rust and CPP and report the metrics as we increase threads. We use Rayon+Rust and CPP+OpenMP for these experiments.

### Machine Details
We benchmark on the NYU HPC crunchy-5 machine with the configuration of 64 cores, 256GB memory. 

### Observations
1. Rust achieves the best peak tokens-per-second highlighting the power of Rust optimizations along with Rayon's multi-processing.
2. CPP scales more gracefully, and improves speeds even as threads/cores > 1. 

### Results
![tokens-per-second](./imgs/tokens-per-second_rust_cpp.png)
![seconds-per-token](./imgs/seconds-per-token_rust_cpp.png)
![speedup](./imgs/speedup_rust_cpp.png)

