# Prior work: silent speech from earbud, jaw and head-worn motion sensors

Ticket: andreaspitzer/subvox#2. Researched 2026-10-31.

Scope: systems that decode silent (mouthed, unvoiced) speech from motion sensors at the ear, jaw or face, plus
adjacent sensor-based silent-speech work (EMG, lip video, articulography) where it tells us about data quantity,
model choice and the **transfer gap** (the difference between voiced and silent articulation).

Evidence labels used below:

- **[paper]**: I read the number in the paper itself (full text or abstract on the publisher or arXiv page).
- **[extract]**: a tool read the full-text PDF and reported the number to me. I did not see the sentence myself.
- **[survey]**: the number comes from a survey or a later paper that cites the work, not from the work itself.
- **snippet**: the number comes from a search-engine result snippet of the paper's abstract page. I did not open the
  page myself.

## Answer

- **Nobody has published silent-speech decoding from commodity earbud head-motion streams.** I found no paper
  that uses AirPods `CMHeadphoneMotionManager` (or any processed head-pose API at about 25 Hz) for speech, jaw
  motion or mouthing. The closest systems use **custom IMUs pressed on the jaw joint (TMJ)**: 100 Hz in MuteIt,
  and the same twin-IMU design in Unvoiced, whose sample rate is not confirmed. Other close systems use **a headphone-mounted IMU at 100 Hz** (Whispering Wearables), **an earbud IMU at 30 Hz for facial
  expression, not speech** (IMUFace), and **face-taped accelerometers at 50 Hz** (Conformer-CTC paper). The
  destination is plausible but not demonstrated. Subvox would be the first to show it.
- **Realistic targets at about 25 Hz are command-scale, not open phrases.** On purpose-built jaw hardware, the
  best published results are 94.8 % top-1 on 100 isolated command words (MuteIt) and WER about 0.07 to 0.15 on
  a small set of phrases with LLM correction (Unvoiced). With a headphone IMU near the ear (the form factor
  nearest to AirPods), IMU-only accuracy was **about 79 to 83 % on only 12 commands**. A 100 to 500 word vocabulary
  with continuous phrases at 25 Hz is beyond anything published. Expect it to need a strong closed-vocabulary
  language model or grammar to work.
- **The signal at 25 Hz is jaw kinematics, not phonemes.** 25 Hz sampling gives a 12.5 Hz Nyquist limit. The
  speech amplitude envelope peaks near 5 Hz (syllable rate), so syllable-rate jaw opening survives. Articulography
  studies keep content up to 20 to 25 Hz, so some fast detail is lost. Vocal vibration (hundreds of Hz and up) is
  lost completely. Jaw-only systems recover syllable count, vowel height class, viseme groups and plosives, not
  full phoneme identity (MuteIt). Plan the "phoneme output" as **coarse phoneme classes plus a lexicon or
  language model**, not as full phoneme recognition. If the measured rate is 50 Hz (reported for AirPods Pro 2 on
  macOS 26; see andreaspitzer/subvox#3), the Nyquist limit is 25 Hz and this limit relaxes.
- **Data per user in the earable work is small: about 1 to 16 minutes, or 5 to 40 repetitions per item.**
  MuteIt: about 1 minute for a per-user vowel/viseme classifier. Unvoiced: about 15 minutes of fine-tuning per
  user gives WER about 0.12. Whispering Wearables: 40 silent repetitions per command (5 for user adaptation).
  IMUFace: 5 minutes. The EMG "closed vocabulary" analogue (67 words, dates and times) used 26 minutes silent plus
  30 minutes voiced from one speaker and reached 3.6 % WER. Open vocabulary EMG needed about 19 hours.
- **Model families that work with small data:** per-word templates or classic classifiers (DTW, SVM, Gaussian
  process) for tens of commands; a phoneme or phoneme-class decoder plus a dictionary search (MuteIt particle
  filter) to scale vocabulary without per-word training; sequence models (BiLSTM or Conformer/Transformer)
  with **CTC** or with **audio-feature targets** for phrases. An **auxiliary phoneme loss** helps (EMG WER 42.2 %
  with it, 51.7 % without). **LLM or language-model rescoring** gives large gains (EMG silent WER 22.2 % to
  12.2 %). I found no earable silent-speech work that uses self-supervised IMU pretraining. The nearest is
  cross-modal contrastive alignment of EMG with audio (MONA).
- **The transfer gap is real and large for naive training.** In EMG, a model trained only on voiced data and
  tested on silent data scored 88.8 % WER, against 3.6 % when audio targets were transferred to the silent
  recordings by DTW alignment (closed vocabulary). In lip-reading, training on normal speech and testing on silent
  speech lost 2.5 to 3.2 points against silent-trained models, and silent speech was the hardest condition overall.
  In articulography, mouthed sentences are **slower, with longer and more numerous movement strokes, and lower peak
  speed and stroke distance** than voiced ones; whispered speech differs little from voiced.
- **The transfer gap in earable motion sensing is barely measured.** Only one earable paper reports a cross-condition
  number: train audible, test silent 91.5 % against train silent, test silent 94.2 % on 12 commands (Whispering Wearables;
  modality not confirmed, probably IMU plus ExG fused, because 94.2 % equals the fused user-dependent figure). Unvoiced trains with audible repetitions (with audio) for its jaw-to-spectrogram model but
  does not report a clear voiced-against-unvoiced gap. This is the main open question the Subvox go/no-go test must
  answer.
- **Practical hints for the gate test:** (1) Labels from voiced sessions are usable only with an alignment step from
  silent to voiced takes (DTW is the standard method; Gaddy and Klein). (2) **Whispered speech transfers to silent
  better than normal speech** in lip-reading (digits 62.8 against 59.7, phrases 65.1 against 61.2), and whispered
  kinematics are close to voiced. Whisper may be a useful bridge condition. (3) Measure duration and peak-speed
  differences between conditions, because those are the known kinematic effects of mouthing.
- **Evidence is thin.** Studies are small (4 to 20 users), per-user, lab-based and mostly prompted commands. Most of
  the earable IMU results come from one research line (Stony Brook, Srivastava, Khanna, Jain et al.: JawSense,
  MuteIt, Unvoiced, Jawthenticate; Whispering Wearables shares the first author), so there is little independent
  replication. Nobody tests against the AirPods sensor-fusion pipeline, whose filtering is not documented.

## Comparison table

Rate = sensor sample rate. UD = user-dependent. "Silent" = mouthed or unvoiced, no sound.

| System (year) | Sensor and placement | Rate | Condition | Vocabulary and task | Result | Per-user data | Model | Evidence |
|---|---|---|---|---|---|---|---|---|
| JawSense (2021) | Low-cost accelerometer at ear/jaw (muscle deformation, vibration) | not found | Silent | 9 phonemes, isolated | 92 % (N=6) | not found | Feature + classifier | [paper] abstract; numbers [survey] |
| MuteIt (2022) | 2 x Mbient MetaMotionR IMU: on TMJ, and behind ear as head-motion reference | 100 Hz | Silent | 100 command words (lexicon 1100 words), isolated | 94.8 % top-1 quiet; 93.8 % head movement, 91.9 % walking, 87 % bus (mean about 91 %) | 5 takes per word for test; about 1 min for per-user vowel/viseme classifier; no per-word training | Gaussian process classifier for vowels/visemes + bidirectional particle filter over phonemes and dictionary | [paper] abstract, [extract] |
| Unvoiced (2024) | Twin IMU as in MuteIt (TMJ + temporal bone), accel +/-2 g, gyro 250 dps | not confirmed (tens of Hz) | Silent at test; training uses audible takes with audio | 20 commands, digits 0-9, 30 phrases, 6-digit codes, 4 tasks | Task completion 93.1-98.6 %; phrase WER 0.07 (seen), 0.11 (unseen phrases), 0.14 (unseen words) | 20 takes per item; about 15 min fine-tune gives WER 0.12 | BiLSTM encoder-decoder with attention, IMU to mel spectrogram, GPT-2 embedding, GPT-4 correction | [extract] |
| Whispering Wearables (2024) | Mbient IMU on headphone near ear, plus ExG electrodes | IMU 100 Hz, ExG 1000 Hz | Silent and audible | 12 commands, isolated | Fused 94.2 % UD; IMU only about 79-83 %; train audible / test silent 91.5 % against 94.2 % matched (modality not confirmed, probably IMU plus ExG fused, because 94.2 % equals the fused UD figure) | 40 silent + 15 audible takes per command; 5 takes for user adaptation | SVM, RBF kernel | [extract] |
| EarCommand (2022) | Near-ultrasound speaker + inward mic in ear canal (deformation); IMU only to trigger | n/a (acoustic) | Silent | 32 words; 25 sentences | WER 10.02 % (words), 12.33 % (sentences), N=12 | not found | not read | [paper] abstract snippet, [survey] |
| EarSSR, ReHEarSSE (2024) | Ultrasound ear-canal deformation | n/a | Silent | Letters, words, phrases; 100 unseen words | 82 % letters, 93 % words; 89.3 % unseen words | not found | not read | [survey] |
| HPSpeech (2023) | Over-ear headphone speaker + mic, acoustic TMJ sensing | n/a | Silent | 8 commands | > 90 %, N=18 | not found | not read | [survey] |
| Magnetic skin (2023) | Magnetometer behind ear | not found | Silent | Phonemes | 93.2 %, N=5 | not found | not read | [survey] |
| IMUFace (2025) | XIAO nRF52840 6-axis IMU on each earbud (12 channels) | **30 Hz** | Facial expression (not speech) | 7 expressions, 51 face landmarks | 2.21 mm landmark error (UD); 2.05 mm with 2 min adaptation | 5 min (1 min gives 2.66 mm) | 1D CNN + Transformer | [paper] |
| ExpressEar (2021) | Earable IMU | not found | Facial action units | 32 action units | 89.9 %, N=12 | not found | Temporal CNN | [survey]/search snippet |
| Face accelerometers, Conformer-CTC (2025) | 6 six-axis accelerometers taped on face | **50 Hz** | Silent | 16 words, 2 English sentences, 8 Chinese phrases | 97.17 % (N=4, incl. 1 laryngectomy patient) | 100 takes per word, 30 per sentence | Conv + local self-attention (Conformer) + CTC | [paper] |
| Gaddy & Klein (2020) | 8-channel facial EMG | 1000 Hz (features 100 Hz) | Silent and voiced, 1 speaker | Closed: 67 words, dates/times. Open: 9828 words, books | Closed silent WER 3.6 % (direct voiced-trained transfer 88.8 %). Open silent 68.0 % (direct 91.2 %) | Closed: 26 min silent + 30 min voiced. Open: 18.6 h total | BiLSTM EMG to audio features + vocoder; silent-to-voiced DTW + CCA target transfer | [paper] |
| Gaddy & Klein (2021) | Same data | same | Silent | Open vocabulary | Silent WER 42.2 % (no phoneme loss 51.7 %) | 19 h | Conv + Transformer + auxiliary phoneme loss | [paper] |
| MONA LISA (2024) | Same data + LibriSpeech audio | same | Silent and voiced | Open vocabulary | Silent WER 22.2 %, 12.2 % with LLM; voiced 8.9 %, 3.7 % with LLM | about 18 h EMG | Cross-modal contrastive (crossCon) + CTC + LLM rescoring | [paper] |
| Petridis et al. (2018) | Lip video | 30 fps | Normal, whispered, silent | 10 digits; 10 phrases; 53 / 39 speakers | Silent: matched 62.2 / 64.4 %; from normal 59.7 / 61.2 %; from whisper 62.8 / 65.1 % | multi-speaker | Bottleneck MLP + BLSTM | [paper] |
| Dromey & Black (2017) | EMA on tongue, jaw, lips | n/a | Voiced, whispered, mouthed | 6 sentences, 20 adults | Mouthed: longer sentences and strokes, more strokes, lower peak speed and stroke distance. Whisper close to voiced | n/a | Kinematic analysis | [paper] abstract |
| Janke, Wand, Schultz (2010) | Surface EMG | n/a | Audible, whispered, silent | Speech recognition | Mode differences visible in EMG; spectral mapping gives up to 11.59 % relative gain on silent | n/a | HMM recognizer + spectral mapping | [paper] abstract |

## Per-source notes

### Earable and head-worn motion sensors (silent speech)

**JawSense (Khanna et al., HotMobile 2021).** First of the Stony Brook line. An ear-worn system that "senses the
muscle deformation and vibration caused by unvoiced speaking to decode the unvoiced phonemes". A citing paper
reports 9 phonemes at 92 % with 6 users. MuteIt later says JawSense "can only recognize individual sounds or
phonemes", which is not enough for words.
Sources: https://dl.acm.org/doi/10.1145/3446382.3448363 ;
https://www.researchgate.net/publication/349571234_JawSense_Recognizing_Unvoiced_Sound_using_a_Low-cost_Ear-worn_System

**MuteIt (Srivastava et al., IMWUT 6(3), 2022).** The most relevant isolated-word result.
- Hardware: two Mbient MetaMotionR IMUs on an ear hook, one on the temporomandibular joint (TMJ), one behind the
  ear on the temporal bone as a head-motion reference. 100 Hz. [extract]
- Key claim from the paper: the jaw is a secondary articulator, so "jaw motion is not distinctive enough for
  unvoiced speech recognition" by itself. [paper]
- Method: segment each word into syllables, find the vowel, the onset viseme group and a terminal plosive, then
  reconstruct the word as a phoneme sequence with a forward and backward particle filter over an English
  dictionary. No per-word training, so the vocabulary can grow. A small per-user model (Gaussian process
  classifier, about 1 minute of data) classifies vowels and visemes. [paper], [extract]
- Results: 20 users, 100 command words, 94.8 % mean top-1 in quiet conditions; 93.8 % with head movement, 91.9 %
  walking, 87 % in a bus; 97 % weighted accuracy for syllable count. [paper]
- Relevance to Subvox: shows what a jaw-only signal can carry (syllables, vowel class, viseme group, plosives).
  It also shows the value of a head-motion reference sensor. AirPods give one fused head pose per bud and no
  separate jaw sensor, so Subvox cannot cancel head motion the same way.
Source: https://dl.acm.org/doi/10.1145/3550281 (PDF: https://dl.acm.org/doi/pdf/10.1145/3550281)

**Unvoiced (Srivastava et al., SenSys 2024).** Phrase-level, closest to the destination's training plan.
- Hardware: the MuteIt twin-IMU design; 3-axis accelerometer (+/-2 g) and 3-axis gyroscope (250 dps). I could not
  confirm the sample rate. The tool extract says the IMU rate is about 10^3 times lower than the 48 kHz audio
  target. [extract]
- Method: cross-modal translation from low-frequency jaw motion to mel spectrograms, with a BiLSTM encoder and an
  attention decoder, a GPT-2 text embedding layer, and GPT-4 prompting at inference to fill gaps and correct
  phrases, filtered by vowel class and syllable count. Training uses synchronized jaw motion, audio and text.
  [extract]
- Data: per user, 20 voiced and 20 unvoiced repetitions of 30 phrases, 20 commands and the digits. About 15 minutes
  of fine-tuning data gives WER 0.12 for 4 of 5 native-language groups; under 16 minutes gives WER < 0.13. [extract]
- Results: 19 users. Task completion 98.6 %, 97.7 %, 94.8 %, 93.1 % on four tasks. Phrase WER 0.07 (training
  phrases), 0.11 (phrases not in training), 0.14 (unseen words). Isolated-word top-1 88 to 93 % by syllable count.
  [extract]
- Transfer gap: the extract says the model trains on audible takes and is tested on unvoiced jaw motion, but I
  did not find a reported number that compares the two conditions. **Confirm in the full text before relying on
  this.**
Source: https://dl.acm.org/doi/10.1145/3666025.3699374 (author copy:
https://people.cs.umass.edu/~phuc/papers/Unvoiced_SenSys24.pdf)

**Whispering Wearables (Srivastava et al., ICMI 2024).** Closest form factor to AirPods, and the only earable
paper with a cross-condition number.
- Hardware: Mbient IMU (+/-4 g, 250 dps) on the lower part of over-ear headphones near the ear, 100 Hz; dry ExG
  electrodes at 1000 Hz on headphones, glasses and face. [extract]
- Task: 12 device, meeting and media commands; 9 participants. [extract]
- Data: 40 silent and 15 audible takes per command per user; 5 takes per command for user adaptation. [extract]
- Model: SVM with RBF kernel. [extract]
- Results: user-dependent 94.2 %, user-independent with 5 adaptation samples 93.5 %; **IMU alone about 79 to 83 %**,
  ExG alone about 84 to 85 %, fused about 94 %. [extract]
- Cross-condition (modality for this table not confirmed): train audible / test silent 91.5 +/- 2.4 %; train
  silent / test audible 92.3 +/- 1.7 %; train audible / test audible 90.8 +/- 5.9 %; train silent / test silent
  94.2 +/- 3.8 %. [extract] On 12 commands the gap is small (about 3 points).
Source: https://www.microsoft.com/en-us/research/wp-content/uploads/2024/11/3678957.3685720.pdf

**EarCommand (Jin et al., IMWUT 2022).** Not an IMU system: an inward speaker sends near-ultrasound into the ear
canal and an inward mic reads the echoes, which change as the canal deforms with articulation. An IMU only detects
activity to trigger recognition. WER 10.02 % on 32 word-level commands and 12.33 % on 25 sentence-level commands
(12 users). It shows that ear-canal deformation carries command-level silent-speech information. AirPods do not
expose the inward mic stream for such sensing (see Subvox README).
Sources: https://pure.ecnu.edu.cn/en/publications/earcommand-hearing-your-silent-speech-commands-in-ear/ ;
DOI https://doi.org/10.1145/3534613 ; survey table https://arxiv.org/html/2506.05720v3

**Other ear-canal and ear-area systems (survey only).** EarSSR (ultrasound ear-canal deformation, 82 % letters,
93 % words/phrases, 50 users), ReHEarSSE (89.3 % on 100 unseen words), HPSpeech (headphone acoustic TMJ sensing,
8 commands, > 90 %, 18 users), magnetic skin behind the ear (93.2 % phonemes, 5 users), QuietSync (IMU + ExG,
5-sample adaptation). I did not read these papers.
Source: Earable survey, Table 7: https://arxiv.org/html/2506.05720v3

**IMUFace (arXiv 2501.02177, 2025).** Not speech, but the best evidence for a **commodity-like earbud IMU at a low
rate**. A XIAO nRF52840 Sense board (6-axis IMU) on each wireless earbud, 30 Hz, 12 channels. It reconstructs 51
facial landmarks during 7 expressions with 2.21 mm mean error (user-dependent, 5 minutes of training data;
1 minute gives 2.66 mm). Mouth landmarks have the largest error. Model: 1D residual CNN plus a 2-layer Transformer,
input includes STFT features. It shows that earbud IMUs at 30 Hz do see facial muscle activity.
Sources: https://arxiv.org/abs/2501.02177 ; https://arxiv.org/html/2501.02177

**ExpressEar (IMWUT 5, 2021).** An earable IMU detects 32 facial action units at 89.9 % average accuracy with
12 users, using a temporal convolutional network. Also evidence that facial muscle motion reaches the ear.
Sources: https://weave.iiitd.edu.in/docs/projects/eear.html ; DOI https://doi.org/10.1145/3478085

**Silent speech sentences with six-axis accelerometers (arXiv 2502.17829, IEEE Sensors Journal, 2025).** Six
accelerometer/gyroscope units on the face, 50 Hz each. 16 words, 2 English sentences and 8 Chinese phrases; 4
participants including 1 laryngectomy patient. 100 takes per word, 30 per sentence, 70/15/15 split, 10x
augmentation by word concatenation and noise. Conformer-style model (3 convolution blocks, 3 local self-attention
blocks) with CTC loss. 97.17 % overall. The paper also reports 95.57 % in another section without explaining the
difference. Relevance: CTC on 50 Hz motion works for a tiny vocabulary with about 100 takes per item.
Source: https://arxiv.org/html/2502.17829v3

**AirPods and `CMHeadphoneMotionManager`.** I searched for papers that use AirPods head-motion data for speech,
jaw or mouth sensing and found none. Apple's API gives attitude, rotation rate and user acceleration from the buds'
inertial sensors; macOS support came with macOS 14, and motion comes from one bud at a time (WWDC23). Developer
reports say yaw can jump when Spatial Audio becomes active. An IPSJ technical report used AirPods data for 5
facial expressions with 8 subjects, but the page was down and I could not read it.
Sources: https://developer.apple.com/videos/play/wwdc2023/10179 ;
https://developer.apple.com/forums/thread/844940 ; https://ipsj.ixsq.nii.ac.jp/records/229381 (not read) ;
Subvox README ("roughly 25 Hz on AirPods").

**Sample rates of other earable IMUs (for context).** The earable survey lists eSense IMU at 20 Hz (another study
used eSense at 50 Hz), EarSet at 50 Hz, OESense accelerometer at 100 Hz, and EarSpy at up to 2000 Hz for detecting
speech vibration. The EarSAVAS dataset has 44.5 hours of synchronized earable audio and IMU for 8 vocal activities.
Sources: https://arxiv.org/html/2506.05720v3 ; https://www.eti.uni-siegen.de/ubicomp/papers/ubi_earcomp2019b.pdf

### Bandwidth at about 25 Hz

- The speech amplitude envelope has a broad modulation peak near 5 Hz (over 25 hours of speech analyzed).
  Source: Ding et al. 2017, https://read.qxmd.com/read/28212857/temporal-modulations-in-speech-and-music
- Articulography studies treat articulator motion as low-pass and filter at 20 to 25 Hz.
  Source: https://arxiv.org/pdf/2006.03107
- Inference (mine, not from a paper): 25 Hz sampling (12.5 Hz Nyquist) keeps syllable-rate jaw motion but loses
  part of the fast articulatory detail. Vocal-fold vibration is far above this band. AirPods sensor fusion may
  smooth further; its filtering is not documented.

### Transfer gap (voiced against silent articulation)

**Gaddy & Klein, "Digital Voicing of Silent Speech" (EMNLP 2020, best paper).** The key method for the destination.
- Data: one speaker, 8 EMG channels at 1000 Hz, features at 100 Hz. Closed vocabulary: 500 utterances of dates and
  times, 67-word vocabulary, 26 min silent and 30 min vocalized. Open vocabulary: 3.6 h silent, 3.9 h vocalized,
  plus 11.2 h vocalized without a silent pair; 18.6 h total.
- Method: align each silent take to its vocalized twin with DTW on EMG features, copy the vocalized audio
  features to the silent frames as targets, refine the alignment with CCA and with predicted audio. Vocalized
  takes are also used in training, with a separate session embedding per speaking mode.
- Transfer gap: a model trained only on vocalized EMG and tested on silent EMG gave 88.8 % WER (closed vocabulary,
  human transcription) and 91.2 % (open vocabulary, automatic). With target transfer: 3.6 % and 68.0 %. The paper
  does not give a signal-level measure of the gap. It cites Toth et al. 2009 for "substantial degradation" when
  vocalized models are applied to silent EMG.
Sources: https://arxiv.org/abs/2010.02960 ; https://ar5iv.labs.arxiv.org/html/2010.02960 ;
https://aclanthology.org/2020.emnlp-main.445

**Gaddy & Klein, "An Improved Model for Voicing Silent Speech" (ACL 2021).** Same 19 h data. Raw EMG into
convolution layers plus Transformer layers, and an auxiliary phoneme-prediction loss (phoneme labels from forced
alignment of the audio). Open-vocabulary silent WER 68.0 to 42.2 %. Ablations: no phoneme loss 51.7 %, LSTM
instead of Transformer 46.0 %, hand-designed features 45.2 %. Human transcription WER 32.3 %. Hardest phoneme
distinctions: nasality (velum) and some vowel pairs.
Sources: https://arxiv.org/abs/2106.01933 ; https://aclanthology.org/2021.acl-short.23/

**MONA LISA (Benster et al., arXiv 2403.05583, 2024).** Same EMG data plus LibriSpeech audio. DTW in a latent space
aligns silent to vocalized takes; cross-modal contrastive loss (crossCon) and CTC; ensemble plus LLM rescoring
(LISA). Silent WER 22.2 % before and 12.2 % after LLM rescoring; vocal EMG 8.9 % and 3.7 %. Prior best (Gaddy 2022)
was 28.8 %. The paper says the silent-against-audio gap fell from 26.5 to 9.9 points. Without LibriSpeech,
crossCon gave no benefit.
Sources: https://arxiv.org/abs/2403.05583 ; https://arxiv.org/html/2403.05583v1

**Janke, Wand, Schultz (Interspeech 2010).** EMG differs between audible, whispered and silent speech. Silent
speech lacks acoustic feedback and relies more on somatosensory feedback, and this is visible in the EMG. A
spectral-mapping front end improves silent recognition by up to 11.59 % relative. Later work from the same group
reports that audible and silent EMG are "quite distinct".
Sources: https://www.isca-archive.org/interspeech_2010/janke10_interspeech.html ;
https://www.scitepress.org/Papers/2012/37872/

**Dromey & Black, "Effects of Laryngeal Activity on Articulation" (IEEE TASLP 25, 2017).** EMA on mid tongue, front
tongue, jaw, lower and upper lip; 20 adults; 6 sentences voiced, whispered and mouthed. Mouthed speech: sentence
duration, stroke count and stroke duration increased; peak stroke speed, stroke distance and hull area decreased.
Whispered speech showed relatively few differences from voiced. The abstract does not split the effects by
articulator. Related work: a 2021 EMA study (11 speakers) found longer consonants and larger lip aperture in
silent speech; Crevier-Buchman et al. 2011 found shorter words and lip hypoarticulation in silent speech (one
speaker). These disagree on direction for lips, so the effect may depend on speaker and task.
Sources: https://scholarsarchive.byu.edu/facpub/7247 ; https://pubs.asha.org/doi/10.1044/2021_JSLHR-20-00266
(abstract via search only) ; https://halshs.archives-ouvertes.fr/halshs-00610870v1 (not read in full)

**Petridis et al., "Visual-Only Recognition of Normal, Whispered and Silent Speech" (ICASSP 2018).** Lip video at
30 fps, 10 digits (53 speakers) and 10 phrases (39 speakers). Silent speech was the hardest mode even when train
and test matched. Silent test accuracy: matched 62.2 % (digits) and 64.4 % (phrases); trained on normal 59.7 % and
61.2 %; **trained on whispered 62.8 % and 65.1 %**. Cross-mode drops up to 8.5 %. Kashiwagi et al. (Interspeech
2023) reduce the normal-to-silent gap with metric learning over shared visemes.
Sources: https://arxiv.org/abs/1802.06399 ; https://ar5iv.labs.arxiv.org/html/1802.06399 ;
https://www.isca-archive.org/interspeech_2023/kashiwagi23_interspeech.pdf (not read in full)

## Gaps and open questions

- No paper measures silent-speech information in AirPods (or any commodity earbud) head-motion output.
- No paper measures the voiced-against-silent gap for jaw or ear IMU signals beyond 12 commands.
- No paper tests 100 to 500 word continuous phrases from ear-area motion alone. Unvoiced is the closest, with a few
  dozen phrases plus LLM correction.
- I could not confirm the Unvoiced IMU sample rate or its exact voiced/unvoiced train-test protocol, nor the
  EarCommand model and per-user data. The ACM full texts were blocked for direct reading.
