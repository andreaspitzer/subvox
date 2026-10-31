# Subvox

Subvox tests whether AirPods head-motion data can turn silently mouthed words into text for one person. The decoder must read silent mouthing only.

## Language

### Recording

**Session**:
One recording of motion, optional audio and segment events under a single condition.
_Avoid_: Run, capture

**Prompted session**:
A session in which the person articulates a given prompt, so the words are known without audio.
_Avoid_: Scripted run, training session

**Passive session**:
A session with audio, recorded during the person's own normal speech and labeled only from that audio.
_Avoid_: Background recording, ambient capture

**Condition**:
The way the person articulates during a session: normal speech, whisper or silent mouthing.
_Avoid_: Mode, style

**Silent mouthing**:
Articulation of words with jaw and lip movement but no voice and no audible breath.
_Avoid_: Subvocalization, miming, lip-sync

**Segment**:
A contiguous span of motion samples that holds one utterance.
_Avoid_: Take, clip, chunk, window

### Decoding

**Template**:
A segment with a known word, stored as a reference for matching.
_Avoid_: Sample, example, recording

### Evaluation

**Transfer gap**:
The difference between the motion of a word in normal speech or whisper and the motion of the same word in silent mouthing.
_Avoid_: Domain shift, condition drift
