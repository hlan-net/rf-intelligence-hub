# Models

AI / LLM model artifacts and related objects.

- Local on-device models: e.g. compiled Hailo `.hef`, or `.gguf` for llama.cpp.
- Conversion / compilation scripts and notes.
- Model cards documenting source model, license, quantization, and target runtime.

**Do not commit large binaries directly to git.** Use Git LFS or an external
artifact store and keep only pointers, checksums, and build instructions here.
Large model weights bloat the repo permanently.

Target model selection (Gemma variant, local vs cloud split) is an open decision —
see [`../ROADMAP.md`](../ROADMAP.md) and requirements §5. Note the requirements
currently reference both "Gemma 3 1B" and "Gemma 4 12B"; this must be reconciled.
