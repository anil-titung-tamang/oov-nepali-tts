# Step 8: Finetuning (bring your own methodology)

This step is intentionally left open-ended. Finetune your target TTS model
using the QC'd recordings from Step 6 as additional training data.

**Input contract:**
- data/processed/recordings_processed/speaker_XX/{OOV,IV}/*.wav
  (48kHz, 16-bit, mono, QC-passed — see step6 notebook)
- data/processed/evaluation_sentences.csv (transcript ? audio mapping)

**Output contract (needed by Step 7):**
- A finetuned model checkpoint (local path or Hugging Face model ID)
- Update the checkpoint reference in 
otebooks/step7_synthesis_and_evaluation.ipynb
  and in README.md — keep this in exactly one place to avoid version
  mismatches (see Known Issues in docs/).

**Notes:**
- Finetuning approach (full finetune vs LoRA/adapters, batch size, learning
  rate, number of epochs, base checkpoint) will vary by architecture and
  team — document your own choices in a notebook or script under this
  folder, e.g. step8_finetune_<your-framework>.ipynb.
- Whatever approach you use, keep a *before-finetune* checkpoint so Step 7
  can run a true baseline-vs-finetuned comparison (this was skipped in the
  current results — see docs/ Known Issues).
