# Cloud transcription with word times

Research for issue #4 (andreaspitzer/subvox). Read on 2026-10-31. All prices are list prices in USD on that date.

Question: which cloud service should label voiced sessions with words and word times? Which forced aligner gives word and phoneme times for prompted sessions? Can a service give phonemes, or must we derive them (G2P)?

## Answer

- **Use AssemblyAI Universal-3.5 Pro on the EU endpoint (`api.eu.assemblyai.com`) to label voiced sessions.** It returns `words[]` with `start`/`end` in milliseconds and a confidence. It costs $0.21 per audio hour, and the EU price is the same as the US price. AssemblyAI does not use files sent to its EU servers for training. Its keyterm list holds up to 1,000 terms (+$0.05/h), so the full closed vocabulary (100–500 words) fits in the prompt.
- **Delete each transcript after you fetch it.** For async jobs, AssemblyAI deletes audio after 24–48 h. It keeps the transcript for 30 days unless you set a TTL or call delete. Account and usage data stay in the US. Do not put personal data in file names.
- **Runner-up: Deepgram Nova-3 on `api.eu.deepgram.com` with `mip_opt_out=true` on every request.** This gives zero retention and processing only in the EU, with no fallback outside the EU. It costs $0.258/h (monolingual). Its keyterm list holds only 100 terms. Without the flag, Deepgram keeps audio and transcripts for training by default.
- **Do not use OpenAI.** Only `whisper-1` returns word timestamps, and OpenAI shuts it down on 2027-02-26. EU processing needs prior approval for zero or modified retention. **Do not use ElevenLabs.** EU residency and zero retention are Enterprise-only. **Google Chirp 3** lists word timestamps under "doesn't support" and warns that turning them on degrades the transcript. **Azure** is a good fit only for its phoneme output (see below).
- **Use cloud transcripts for text, not as the source of truth for timing.** No vendor publishes how accurate its word timestamps are. Take the cloud transcript, correct it against the vocabulary, then force-align it with MFA. Then voiced and prompted sessions get the same timing method and the same phone set.
- **Use Montreal Forced Aligner (MFA 3.4.x) for word and phoneme times.** It aligns a known transcript with a pronunciation dictionary and writes TextGrid/JSON/CSV. Pretrained CC BY 4.0 models and dictionaries exist for US English (ARPA and MFA phone sets) and German (MFA phone set). WhisperX and MMS give word or character times only, not phonemes.
- **No general transcription service returns phonemes. Derive them.** For a closed vocabulary, keep one hand-checked lexicon (word to phones) in the repo. Seed it from the MFA dictionary, fill gaps with `mfa g2p`, and check each entry by ear. The only cloud phoneme source is Azure pronunciation assessment, which returns phonemes with offset and duration for a reference text. It gives phoneme names only for `en-US`. Use it at most as a cross-check.
- **Whispered speech:** no vendor documents quality on whispered speech. Primary benchmarks show a large accuracy drop. Google STT on wTIMIT went from 11.55% WER (normal) to 44.70% WER (whispered) [WESPER, CHI '23]. So label from **voiced** sessions, and use prompted scripts plus forced alignment for whispered takes.
- **Convert `audio.caf` before upload.** OpenAI does not list CAF. Deepgram and Azure list WAV and FLAC. Convert to 16 kHz mono WAV or FLAC on the Mac first. Keep the original CAF locally.
- **Cost is not a deciding factor.** 20 h of voiced audio costs about $5 on AssemblyAI ($0.26/h with keyterms) or about $7 on Deepgram ($0.336/h with keyterms).

## Comparison table

Pricing pages read 2026-10-31. "/h" means per audio hour. A minute price × 60 gives the hourly price.

| Service and model | Word timestamps | Timestamp accuracy published? | Whispered speech documented? | Price /h (batch, list) | Training use by default | Retention | EU processing | GDPR terms |
|---|---|---|---|---|---|---|---|---|
| **AssemblyAI** Universal-3.5 Pro | Yes: `words[]` start/end in ms, confidence [A1] | No | No [A5] | $0.21; keyterms +$0.05 (≤1,000 terms); EU same price [A2] | Some files can be used. Not when you use EU servers, have a BAA, or opt out. Free users cannot opt out [A3][A4] | Async audio: deleted after 24–48 h. Transcript: 30 days, or TTL down to 1 h, or on delete request [A3] | `api.eu.assemblyai.com`; audio and transcripts in the EU; account/usage data in the US; file names may go to US subprocessors [A6] | DPA with EU SCCs or DPF; AssemblyAI is processor [A7] |
| **Deepgram** Nova-3 | Yes: `words[]` start/end in s, confidence [D1] | No | No [D7] | $0.258 mono, $0.312 multi; keyterms +$0.078 (≤100 terms) [D2][D3] | **Yes**: audio and transcripts kept for MIP. Opt out per request with `mip_opt_out=true` [D4] | Opted out: kept only while the request runs ("zero data retention"). Usage metadata: 90 days [D4] | `api.eu.deepgram.com`; no fallback outside the EU; full in-region only with opt-out [D4][D5] | Privacy policy says DPAs and SCCs are signed with customers; no public DPA found [D6] |
| **OpenAI** whisper-1 | Yes, but only `whisper-1` supports `timestamp_granularities[]`; shutdown 2027-02-26 [O1] | No | No | $0.36 (`whisper-1`); `gpt-transcribe` $0.27 has no documented word times [O2] | No [O3] | Abuse logs up to 30 days; ZDR needs prior approval [O3] | `eu.api.openai.com`; needs ZDR or modified abuse monitoring plus an amendment; +10% for models released on or after 2026-03-05 [O3][O2] | DPA; EEA customers contract with OpenAI Ireland Ltd.; SCCs [O4] |
| **ElevenLabs** Scribe v2 | Yes: word or **character** granularity [E1][E2] | No (marketing says "precise") | No | $0.22; keyterms +$0.05 (≤1,000 terms) [E3] | Yes for non-Enterprise, unless you turn off "Improve the models for everyone" [E4] | Kept by default; zero retention (`enable_logging=false`) is Enterprise-only [E5] | EU isolated environment is Enterprise-only; processing stays in the EU only with zero retention [E6] | DPA with SCCs, UK Addendum, DPF [E7] |
| **Google** Chirp 3 (STT V2) | Listed under "doesn't support". Can be turned on, but "some transcription degradation is expected". Batch limited to 20 min with word times [G1] | No | No vendor data. Third-party: 44.70% WER whispered vs 11.55% normal on wTIMIT [W1] | $0.96 standard; $0.18 dynamic batch [G2] | No; data logging is opt-in for a discount [G3] | Not logged by default [G3] | `eu` multi-region (`eu-speech.googleapis.com`); data at rest and in use stays in Europe [G1][G4] | Cloud Data Processing Addendum (not read for this note) |
| **Azure** Speech (fast / batch) | Yes: `words[]` with `offsetMilliseconds`/`durationMilliseconds` in fast transcription [Z1] | No | No | $0.36 fast, $0.18 batch, $1.00 real-time (West Europe) [Z2] | Not stated on the page we read; real-time, fast and pronunciation assessment keep no data [Z3] | Real-time, fast, pronunciation assessment: no storage. Batch: customer storage or TTL [Z3] | Any EU region; EU Data Boundary covers regional Azure services [Z4] | Microsoft Products and Services DPA [Z4] |
| **Azure** pronunciation assessment | Word, syllable and **phoneme** offset/duration (100 ns units) for a reference text [Z5] | No | No | Billed at the STT rate (≈ $1.00/h real-time, West Europe) [Z6][Z2] | as above | No storage [Z3] | as above | as above |

## Details

### What the labels need

The decoder needs, for each voiced or prompted session: the word sequence, a start and end time for each word, and a phone sequence with phone times. All times must be on the `motion.csv` clock. `meta.json` already records the audio start offset (`Subvox/Recording/SessionRecorder.swift`). Word times from a cloud service are relative to the start of the uploaded file. Add the audio offset from `meta.json` to put them on the motion clock.

### AssemblyAI

- Word objects carry `text`, `start`, `end` (milliseconds) and `confidence`. Source: [A1] https://www.assemblyai.com/docs/pre-recorded-audio/api-reference/transcripts/get.md
- Pricing (read 2026-10-31): Universal-3.5 Pro $0.21/h, Universal-2 $0.15/h. Keyterms prompting +$0.05/h, up to 1,000 terms on Universal-3.5 Pro. "EU region is the same price as US." $50 free credit. Source: [A2] https://www.assemblyai.com/pricing
- Training: "We will not use files you submit for model training if you are subject to a Business Associate Addendum, are utilizing our European servers, or if you have opted out from model training." Async retention: uploaded audio is deleted 24–48 h after upload. A transcript with no TTL is deleted after 30 days, or earlier on a delete request. Source: [A3] https://www.assemblyai.com/docs/data-retention-and-model-training.md
- Opt-out is a self-serve dashboard toggle for paid users. Free users cannot opt out. Source: [A4] https://assemblyai.com/docs/faq/how-to-opt-out-of-data-sharing-for-our-model-improvement-program.md
- Languages: Universal-3.5 Pro covers 18 languages, including English and German. Source: https://www.assemblyai.com/docs/pre-recorded-audio/universal-3-5-pro.md
- Whispered speech: the docs index has no page on whispered speech. Source: [A5] https://www.assemblyai.com/docs/llms.txt
- EU endpoint `api.eu.assemblyai.com` processes audio and transcripts in Frankfurt, Stockholm or Ireland. Account data and usage data are stored in the US. File names may go to CrowdStrike and Datadog in the US. Source: [A6] https://www.assemblyai.com/docs/pre-recorded-audio/select-the-region.md
- DPA: "Customer is the Controller and AssemblyAI is a Processor". Transfers rely on the DPF or SCCs. Subprocessors get 10 days' notice. Source: [A7] https://www.assemblyai.com/legal/data-processing-addendum

### Deepgram

- Pre-recorded responses carry `words[]` with `start`, `end` (seconds), `confidence` and `punctuated_word`. Source: [D1] https://developers.deepgram.com/docs/pre-recorded-audio.md
- Pricing (read 2026-10-31): Nova-3 monolingual $0.0043/min, multilingual $0.0052/min. Keyterm prompting $0.0013/min. $200 free credit. The page does not list a separate EU price. Source: [D2] https://deepgram.com/pricing
- Keyterm prompting takes up to 100 terms on Nova-3 and Flux. Separators such as commas fail silently: repeat the `keyterm` parameter for each term instead. Source: [D3] https://developers.deepgram.com/docs/keyterm.md
- Data: "Requests participate in the Model Improvement Program (MIP) by default, and Deepgram retains them. Retained data may be processed outside your region." With `mip_opt_out=true`, "Deepgram does not store your audio, text, transcripts, or synthesized audio after the response is returned". Usage metadata stays retrievable for 90 days. "A full in-region guarantee requires both a regional endpoint and `mip_opt_out=true`." Source: [D4] https://developers.deepgram.com/trust-security/your-data.md
- EU endpoint `api.eu.deepgram.com` handles Nova and Flux STT, but not Whisper models. "Requests to the EU endpoint are never routed outside the EU: if the region is unavailable, requests fail rather than fall back to another region." Source: [D5] https://developers.deepgram.com/reference/regional-endpoints.md
- GDPR: the privacy policy says Deepgram is a processor, signs DPAs and signs SCCs. The subprocessor page says to request a DPA from security@deepgram.com. We found no public DPA text. Sources: [D6] https://deepgram.com/privacy and https://deepgram.com/privacy/subprocessors
- Whispered speech: the docs index has "Whisper" pages only for the OpenAI Whisper model, not for whispered speech. Source: [D7] https://developers.deepgram.com/llms.txt
- Languages: Nova-3 lists German and English in monolingual and multilingual modes. Source: https://developers.deepgram.com/docs/models-languages-overview.md

### OpenAI

- The guide lists `gpt-transcribe` as the recommended model. It marks `whisper-1`, `gpt-4o-transcribe`, `gpt-4o-mini-transcribe` and `gpt-4o-transcribe-diarize` as deprecated, with shutdown on February 26, 2027. "The `timestamp_granularities[]` parameter is only supported for `whisper-1`." Upload formats: mp3, mp4, mpeg, mpga, m4a, wav, webm; maximum 25 MB. Source: [O1] https://developers.openai.com/api/docs/guides/speech-to-text
- The API reference does not say whether `gpt-transcribe` returns word times. Source: https://developers.openai.com/api/reference/resources/audio/subresources/transcriptions/methods/create
- Pricing (read 2026-10-31): Whisper $0.006/min, `gpt-transcribe` $0.0045/min, `gpt-4o-mini-transcribe` $0.003/min. "Regional processing (data residency) endpoints are charged a 10% uplift for models released on or after March 5, 2026." Source: [O2] https://developers.openai.com/api/docs/pricing
- Data: API data is not used for training by default. Abuse logs are kept up to 30 days. `/v1/audio/transcriptions` is ZDR-eligible, but ZDR needs prior approval. EU processing (`eu.api.openai.com`) supports `/v1/audio/transcriptions`, but it "Requires Zero Data Retention, Modified Abuse Monitoring, Private Retention with PSP, or Safety Retention", plus a Modified Retention amendment. Source: [O3] https://developers.openai.com/api/docs/guides/your-data
- DPA: customers in the EEA or Switzerland contract with OpenAI Ireland Ltd. Transfers use SCCs or adequacy. Source: [O4] https://cdn.openai.com/pdf/openai-data-processing-addendum.pdf (the HTML page https://openai.com/policies/data-processing-addendum/ returned 403 to our fetcher, so we did not read the full text).

### ElevenLabs

- Scribe v2 lists "Precise word-level timestamps", keyterm prompting up to 1,000 terms, and 90+ languages. Source: [E1] https://elevenlabs.io/docs/overview/capabilities/speech-to-text.md
- `timestamps_granularity`: "'word' provides word-level timestamps and 'character' provides character-level timestamps per word". `enable_logging=false` turns on zero retention, for Enterprise only. Source: [E2] https://elevenlabs.io/docs/api-reference/speech-to-text/convert.md
- Pricing (read 2026-10-31): Scribe v2 $0.22/h, keyterms +$0.05/h, Realtime $0.39/h. Source: [E3] https://elevenlabs.io/pricing/api
- Training: "ElevenLabs uses certain data you provide to us to improve the quality of our audio models for everyone." "By default, we don't train on any data from our Enterprise customers". Others opt out with a toggle. Source: [E4] https://elevenlabs.io/docs/help-center/legal/is-my-data-used-to-improve-eleven-labs-ai-models.md
- Retention: "By default, ElevenLabs retains data". Zero Retention Mode covers `/v1/speech-to-text/` and is for Enterprise only. Source: [E5] https://elevenlabs.io/docs/eleven-api/resources/zero-retention-mode.md
- EU residency: "Data residency is an Enterprise feature." Data is stored in the EU, but "processing may nevertheless occur outside of the selected location". Processing stays in the EU only with Zero Retention Mode and the API. It uses a separate account at `api.eu.residency.elevenlabs.io`. Source: [E6] https://elevenlabs.io/docs/overview/administration/data-residency.md
- DPA: ElevenLabs is processor; transfers use EEA SCCs (Irish law), the UK Addendum and the DPF. Source: [E7] https://elevenlabs.io/dpa

### Google Cloud Speech-to-Text (Chirp 3)

- Chirp 3 lists "Word-level timestamps" under "Chirp 3 doesn't support the following features", with the note: "can be optionally enabled, which some transcription degradation is expected. Available only in Speech.Recognize and Speech.BatchRecognize". "Word-level confidence scores": "The API returns a value, but it isn't truly a confidence score." BatchRecognize allows "up to 20 minutes with word-level timestamp enabled". Locations: `us` and `eu` multi-region. Adaptation takes up to 1,000 phrases. Source: [G1] https://docs.cloud.google.com/speech-to-text/docs/models/chirp-3
- Pricing (read 2026-10-31), V2 API: standard recognition $0.016/min for the first 500,000 min. Dynamic batch $0.003/min. Chirp counts as a "Standard" model. Source: [G2] https://cloud.google.com/speech-to-text/pricing
- "By default, Cloud Speech-to-Text does not log customer audio data or transcripts." Data logging is an opt-in program with discounted prices. Source: [G3] https://docs.cloud.google.com/speech-to-text/docs/data-logging
- EU endpoint `eu-speech.googleapis.com`: "your data at-rest and in-use will stay within the continental boundaries of Europe or the USA". Source: [G4] https://docs.cloud.google.com/speech-to-text/docs/endpoints

### Azure AI Speech

- Fast transcription returns `words[]` with `offsetMilliseconds` and `durationMilliseconds`. Files must be under 5 h and 500 MB. Formats include WAV, MP3, OPUS/OGG and FLAC. Phrase lists are supported from API version 2025-10-15. Source: [Z1] https://learn.microsoft.com/en-us/azure/ai-services/speech-service/fast-transcription-create
- Pricing (read 2026-10-31) from the Azure Retail Prices API, region `westeurope`: "S1 Speech To Text" $1.00/h, "Fast Transcription Speech To Text" $0.36/h, "S1 Speech to Text Batch" $0.18/h. The HTML pricing page loads prices by script, so we used the API. Source: [Z2] https://prices.azure.com/api/retail/prices?$filter=productName eq 'Azure Speech' and armRegionName eq 'westeurope'
- "When doing real-time speech to text, fast transcription, pronunciation assessment, and speech translation, Microsoft does not retain or store the data provided by customers." Batch output stays in customer storage, or in Microsoft storage with a TTL. Source: [Z3] https://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/speech-service/speech-to-text/data-privacy-security
- EU Data Boundary: "regional services that a customer deploys in an EU Data Boundary region will be in-scope". Austria is an EU Data Boundary country. Source: [Z4] https://learn.microsoft.com/en-us/privacy/eudb/eu-data-boundary-learn

### Phoneme output from a service: Azure pronunciation assessment only

- Granularity `Phoneme` returns scores "on the full text, word, syllable, and phoneme level". Each element has `Offset` and `Duration` in 100-nanosecond units. Phoneme alphabet: IPA or SAPI. Phoneme names exist only for `en-US` (IPA and SAPI) and `zh-CN` (SAPI). `ReferenceText` is optional: with it you get a scripted assessment. For audio over 30 s, use continuous mode. Source: [Z5] https://learn.microsoft.com/en-us/azure/ai-services/speech-service/how-to-pronunciation-assessment
- "Usage of pronunciation assessment costs the same as speech to text for Standard or commitment tier pricing". Prosody costs extra. Source: [Z6] https://learn.microsoft.com/en-us/azure/ai-services/speech-service/pronunciation-assessment-tool
- Limits for our use: it scores pronunciation against a native-speaker model and is not built as an aligner. German results come without phoneme names. It is real-time priced ($1.00/h). Use it at most to cross-check MFA phone times on a small en-US sample.
- The other five services return words (ElevenLabs also characters), not phonemes. Their response schemas (sources [A1], [D1], [O1], [E2], [G1], [Z1]) show no phoneme field.

### Forced aligners

**Montreal Forced Aligner (recommended)**

- What it does: "Forced alignment is a technique to take an orthographic transcription of an audio file and generate a time-aligned version using a pronunciation dictionary to look up phones for words." It runs several Kaldi passes: monophone, triphone, LDA+MLLT, then speaker adaptation. Source: https://montreal-forced-aligner.readthedocs.io/en/latest/user_guide/index.html
- Command: `mfa align CORPUS_DIRECTORY DICTIONARY_PATH ACOUSTIC_MODEL_PATH OUTPUT_DIRECTORY`. Output formats: `long_textgrid` (default), `short_textgrid`, `json`, `csv`. `--g2p_model_path` handles OOV words. `--fine_tune` adds a boundary fine-tuning pass. Since 3.4 there is also `mfa align_hf` with Hugging Face models that bundle dictionary and G2P. The 4.0 roadmap renames the old commands to `*_legacy`. Sources: https://montreal-forced-aligner.readthedocs.io/en/latest/user_guide/workflows/alignment.html and https://montreal-forced-aligner.readthedocs.io/en/latest/changelog/index.html
- Latest release: v3.4.3, 2026-10-08. Install: `conda install -c conda-forge montreal-forced-aligner`. Sources: https://github.com/MontrealCorpusTools/Montreal-Forced-Aligner/releases and the repo README.
- Models (all CC BY 4.0): English (US) ARPA v3.0.0 and English MFA v3.1.0 acoustic models; German MFA v3.0.0. Dictionaries: English (US) ARPA, English (US) MFA, German MFA, German Prosodylab. Sources: https://mfa-models.readthedocs.io/en/latest/acoustic/index.html and https://mfa-models.readthedocs.io/en/latest/dictionary/index.html
- G2P: `mfa g2p` uses Pynini models. Per-utterance G2P "is largely not recommended", because it takes only the top guess per word in isolation and gets homographs wrong. For a closed vocabulary this does not matter, because we check each entry by hand. Source: https://github.com/MontrealCorpusTools/Montreal-Forced-Aligner/blob/main/docs/source/user_guide/workflows/dictionary_generating.rst
- Accuracy: the Interspeech 2017 paper reports that MFA "performs well relative to two existing open-source aligners with simpler architecture (Prosodylab-Aligner and FAVE)". McAuliffe et al., https://www.isca-archive.org/interspeech_2017/mcauliffe17_interspeech.html. We did not read the paper body for boundary-error numbers.

**WhisperX**

- Gets word times by aligning Whisper output with a phoneme-based wav2vec2 CTC model. Default align models exist for en, fr, de, es and it. Words with characters the align model does not know (for example numbers) get no times. It returns words (and optionally characters), not phonemes. BSD-2-Clause. Latest release v3.8.6 (2026-05-25). Source: https://github.com/m-bain/whisperX
- Word timing benchmark (200 ms collar, exact word match), precision/recall %: WhisperX 84.1/60.3 on AMI and 93.2/65.4 on Switchboard. Plain Whisper (DTW over attention): 78.9/52.1 and 85.4/62.8. Source: Bain et al., "WhisperX: Time-Accurate Speech Transcription of Long-Form Audio", https://arxiv.org/abs/2303.00747. This is the only primary number we found for Whisper-style word timing. It suggests `whisper-1` word times are rough.
- Use for us: a free local option to transcribe and align voiced sessions without a cloud service. It is not a phoneme aligner.

**MMS forced alignment (torchaudio `MMS_FA`)**

- Acoustic model "trained with 23,000 hours of audio from 1100+ languages". It aligns uroman-romanized **characters** (a–z, apostrophe), giving word or character times, not phonemes. Torchaudio warns: "The APIs described in this tutorial are deprecated in 2.8 and will be removed in 2.9." Source: https://docs.pytorch.org/audio/stable/tutorials/forced_alignment_for_multilingual_data_tutorial.html. Paper: Pratap et al., "Scaling Speech Technology to 1,000+ Languages", https://arxiv.org/abs/2305.13516
- Use for us: none. MFA covers English and German with phones, and the torchaudio API is going away.

**Phoneme CTC alternative (not recommended now)**

- `facebook/wav2vec2-xlsr-53-espeak-cv-ft` "outputs a string of phonetic labels" and was fine-tuned on Common Voice. Apache-2.0. Paper: Xu, Baevski, Auli, "Simple and Effective Zero-shot Cross-lingual Phoneme Recognition", arXiv 2109.11680. Source: https://huggingface.co/facebook/wav2vec2-xlsr-53-espeak-cv-ft. It could be a CTC phone aligner, but we would have to build the tooling ourselves. MFA already gives phone tiers.

### G2P sources for the lexicon

- MFA dictionaries and `mfa g2p` (see above). The phone set must match the acoustic model: ARPA for `english_us_arpa`, MFA phones for `english_mfa` and `german_mfa`.
- eSpeak NG "Can translate text into phoneme codes" and "supports more than 100 languages and accents". GPL-3.0-or-later, so keep it as an offline tool and do not ship it in the app. Source: https://github.com/espeak-ng/espeak-ng

### Whispered speech: evidence

- No vendor (AssemblyAI, Deepgram, ElevenLabs, OpenAI, Google, Azure) documents accuracy on whispered speech. We checked the doc indexes and model pages cited above.
- Rekimoto, "WESPER: Zero-shot and Realtime Whisper to Normal Voice Conversion for Whisper-based Speech Interactions", CHI 2023: Google Cloud STT on wTIMIT gave 11.55% WER on normal speech and 44.70% WER on whispered speech, and 26.68% after voice conversion. Source: [W1] https://arxiv.org/html/2303.01639
- Farhadipour, Asadi, Dellwo, "Leveraging Self-Supervised Models for Automatic Whispered Speech Recognition", ICCKE 2024: an OpenAI Whisper baseline had 18.8% WER and 4.24% CER on whispered speech. The abstract does not give the normal-speech WER. Source: https://arxiv.org/abs/2407.21211
- MFA acoustic models are trained on voiced speech. We found no primary benchmark of MFA boundary accuracy on whispered speech. Treat whispered alignments as lower quality, and spot-check them.

## Suggested pipeline (input for the spec)

1. Voiced session: convert `audio.caf` to 16 kHz mono FLAC. Send it to AssemblyAI EU with `keyterms_prompt` = the vocabulary. Store the JSON locally next to the session. Delete the transcript on the server.
2. Normalize the transcript to vocabulary tokens. Flag words that are not in the vocabulary, or that have low confidence, for review.
3. Prompted session: use the script as the transcript and skip step 1.
4. Run `mfa align` with the repo lexicon and the matching acoustic model. Write JSON with word and phone tiers.
5. Shift all times by the audio offset in `meta.json` to put them on the motion clock.
6. Check: compare AssemblyAI word times with MFA word times. A large gap marks a bad alignment or a bad transcript.

## Open points

- Language of the vocabulary (English, German, or both) sets the MFA model and phone set. This note does not decide it.
- Deepgram's DPA text is not public. Ask security@deepgram.com if Deepgram becomes the primary service.
- AssemblyAI and Deepgram keyterm limits and prices change often. Check them again before you commit.
- Firecrawl hit its keyless daily limit during this research. We read the remaining pages with WebFetch and curl.
