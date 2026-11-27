# Inference Architecture

**Status:** Proposed; hardware and model compatibility unverified.

llama.cpp is the initial local inference candidate. Hosted providers use provider-neutral adapters. Agents may use independently selected models. GPU generation requires explicit admission control.

The GTX 1650 is a primary experiment candidate and the P1000 is secondary and experimental. Their VRAM MUST NOT be pooled.
