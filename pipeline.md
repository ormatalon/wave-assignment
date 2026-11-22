# Wave pipeline

Assumptions live in `assumptions.md`.

**Scope assumption:** the assignment is only about detecting anomalies in the signal. Ongoing trends in the machine are out of scope, so each iteration scores only its own 10-second segment.

## Flow

```
4 mics (tetrahedral) -> beamforming -> STFT -> mel (128) -> log
                                                  |
                         +------------------------+------------------------+
                         |                                                 |
               detection: is something wrong?               explanation: which component?
               (autoencoder / BEATs + kNN)                  (per-band robust z-score)
                         |                                                 |
                         +----> threshold on this segment -> alert
```

## Stage 1: beamforming

- **What:** combine the 4 microphones to point at the machine and its components (delay-and-sum, or MVDR to actively cancel noise from other directions).
- **Why the tetrahedral layout:** the mics aren't in one plane, so the array gets 3D direction (horizontal angle and height) with no front-back confusion, and can point at a specific component.
- **Why not score each mic separately:** scoring each mic separately can't cancel outside noise, because a nearby hammer reaches all four mics and every channel flags it as an anomaly. Beamforming uses the differences between the mics to point at the machine, so it suppresses outside sounds before scoring.

## Stage 2: features

- **STFT** on the beamformed signal.
- **Mel filterbank:** a fixed matrix (nothing learned) that merges ~1,000 STFT bins into 128 bands spaced the way the ear hears pitch, then log.
- **Sample rate:** see `assumptions.md`. The model runs at 16 kHz, so content above 8 kHz is lost; keep a cheap high-band energy feature for bearing tones.
- **Feature extraction from the mel bands, three options:**
  - **Pretrained models (frozen):** BEATs (a transformer) or PANNs (a CNN). Strongest, no training, but lose interpretability.
  - **Models trained on our own data:** the autoencoder's bottleneck, or the auxiliary-classification CNN. Fit the edge; still lose interpretability.
  - **Hand-made features:** band statistics, spectral flux (clicks and bangs), harmonic tracking (shifting motor tones). Each means something physical, so they're explainable, but they only catch the faults you chose to measure.
- **Scorer at the end:** kNN distance, or alternatives such as Isolation Forest or Local Outlier Factor (density-aware kNN, better when normal states form clusters of different spread), on whichever features are chosen.

## Stage 3: detection and explanation

- **Per-band robust z-score:** each band's energy vs its healthy calibration level, using median and MAD. Says *which* band, and so which component. Explainable, no training.
- **Autoencoder on log-mel (DCASE baseline):** reconstruction error is the anomaly score. Small enough for a microcontroller; error per band also points to a location. Risk: can reconstruct a fault too well and miss it.
- **Per-band embeddings:** one small embedding model per component's frequency band (motor/mains harmonics, bearing tones, gear mesh, broadband), so each component gets its own score by design. This solves the interpretability loss of embeddings: the band whose score rises names the component. Trained on normal data only with auxiliary classification (device ID, mic ID or augmentation labels) and an ArcFace/AdaCos loss. Small CNNs fit the edge. Needs the component frequencies per machine; see `strategy_per_band_embeddings.md`.
- **BEATs + kNN:** frozen pretrained embeddings, scored by distance to healthy calibration embeddings. Strongest, no training. Loses interpretability; needs at least a Raspberry Pi-class board, or can run in the cloud since the device can send audio (at the cost of latency).
- **Combination:** small device: the autoencoder decides on the alert and the per-band score explains it. Strong device or cloud: the autoencoder and BEATs vote, the per-band score explains, and an extreme per-band score can alert on its own. Normalize every score to a z-score against calibration before combining.

### Option: auxiliary classification

```
TRAINING (normal data only)
healthy clips from many machines
        |
        v
embeddings (pretrained model)
        |
        v
k-means -> pseudo-labels (machine categories)   <- or device ID / mic ID if known
        |
        v
small CNN + ArcFace/AdaCos: learn to tell the categories apart
        |
        v
one tight cluster per category

IN USE (on the device)
new clip -> CNN embedding -> distance to its own cluster -> anomaly score
```

- The classification is only a training trick; in use, the device already knows its machine.
- To tell machines apart, the model must learn fine detail (harmonics, bearing tones), which a fault distorts.
- A fault lands outside its cluster, so the distance is the anomaly score.
- With no machine labels, k-means on embeddings gives pseudo-labels (e.g. rotating vs. not, by size).
- Risk: clusters may group by room or mic placement, not machine type; check what each cluster holds.
- Needs several machines; with one machine there is nothing to tell apart.
- Small CNN fits the edge; loses interpretability unless combined with per-band scores.

## Additional ideas (future work): time and trends

- **Two baselines:** a fixed one from calibration (catches slow drift) and a rolling one from recent segments (catches sudden change).
- **Alert on a run of segments,** not a single one, so one-off sounds like a hammer don't trigger it.
- **Slow drift:** a separate trending alert per component (CUSUM or Page-Hinkley). If all bands move together, suspect the mic or the room, not the machine.
- **Threshold:** set from a daily false-alarm budget per machine, not from accuracy.

## Evaluation

Evaluation uses external sources only, with precision-recall as the main metric, reported per machine type.

- **Data:** public datasets with real labelled faults (MIMII, DCASE Task 2), plus synthetic faults injected into their healthy clips.
- **Main metric:** PR-AUC. ROC AUC looks too good with rare faults, because many false alarms still show as a tiny rate.
- **Single number at the chosen threshold:** F-beta with beta 0.5, so precision weighs more, since every false alarm costs the operator's trust.
- **Threshold:** picked from the PR curve, e.g. the highest precision that still catches 90% of faults. Caution: the fault rate in MIMII/DCASE is far higher than on a real site, so on-site precision will be lower.
- **Splits:** by machine or recording session, never by clip, so the test is on recordings the model hasn't heard.
- ★ To check: **Detectability:** inject faults at decreasing loudness to see how quiet a fault can get before it's missed.
- **Per machine type:** report metrics separately for each type (pumps, fans, valves). Combine with a harmonic mean, which drops sharply if any one type is weak.
