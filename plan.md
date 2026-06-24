# Lab Execution Plan

## Goal
Complete the Day 20 llama.cpp lab with the minimum local install footprint, keep heavy tools off `C:` where possible, and store the model files in this repo.

## Findings From The Repo

- `02-llama-cpp-server/README.md` and `rubric.md` show that the native `/metrics` requirement needs `make build-llama`.
- `00-setup/windows-setup.ps1` and `00-setup/linux-setup.sh` both default to CPU-only `llama-cpp-python` unless CUDA is explicitly enabled.
- `00-setup/detect-hardware.py` recommends `Qwen2.5-1.5B-Instruct (Q4_K_M)` on this machine.
- `00-setup/download-model.py` expects both the primary `Q4_K_M` model and the smaller comparison quantization.

## Current Setup Strategy

- Keep Python dependencies in the WSL venv at `.venv-wsl` for runnable lab steps.
- Use WSL2 for the native build toolchain so `cmake`, `make`, `g++`, and related packages live in the WSL virtual disk under `D:\Tools\WSL`.
- Build `llama.cpp` on the WSL ext4 filesystem, then copy the runtime `build/bin` artifacts back into the repo path that the lab scripts expect. The full source build on `/mnt/d` was too heavy for this machine.
- Keep the model files in `models/` in this repo.
- Avoid CUDA Toolkit unless a later step explicitly requires GPU building.

## Steps

1. Repo-local `venv` is ready.
2. WSL2 Ubuntu is imported under `D:\Tools\WSL`.
3. Linux build stack is installed inside WSL:
   - `build-essential`
   - `cmake`
   - `git`
   - `python3-venv`
   - `python3-pip`
   - `pkg-config`
4. Required GGUF model files are present in `models/`.
5. `hardware.json` and `models/active.json` are written.
6. Completed: install repo Python dependencies and `llama-cpp-python` in WSL.
7. Completed: run `01-llama-cpp-quickstart` and capture the benchmark output.
8. Completed: build the native `llama-server` from source on WSL ext4 and verify `make serve-native` can start it.
9. Completed: finish `submission/REFLECTION.md` after the remaining metrics notes are captured.

## Notes

- The lab does not require a CUDA Toolkit for the core path.
- The native metrics and continuous-batching observations do require the source-built `llama-server`.
- If disk space becomes tight, the WSL distro can be removed later from the `D:\Tools\WSL` location without affecting the repo files.
