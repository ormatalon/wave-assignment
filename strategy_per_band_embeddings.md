# Strategy: per-band embeddings (per-component scoring by design)

## Idea
Train one small embedding model per component frequency band, so each component gets its own anomaly score by construction instead of one score per machine.

## Flow
```
4-mic audio (10 s window)
    │
    ▼
Beamforming toward the machine  →  log-mel / STFT
    │
    ▼
Split into component bands
(motor/mains harmonics · bearing tones · gear mesh · broadband)
    │
    ├─► Band 1 → small CNN encoder → embedding → score₁
    ├─► Band 2 → small CNN encoder → embedding → score₂
    └─► Band k → small CNN encoder → embedding → scoreₖ
    │
    ▼
Health score per component  →  persistence rule  →  alert
```

## Training (normal data only)
- Auxiliary classification per band: labels = device ID (machine), mic ID, or self-supervised augmentation labels (pitch shift / time stretch).
- Loss: ArcFace / AdaCos, so normal clips form tight clusters on a hypersphere.
- Score: 1 − cos(θ) to the class centre, or kNN distance to stored normal embeddings.

## Why
- Per-component health score is what the brief asks for (לכל רכיב).
- Interpretability: a high score already names the band, so the component.
- External noise (e.g. hammering) is broadband; requiring the anomaly to sit in a component's own band reduces false alarms.
- Small CNNs (MobileNet / MobileFaceNet scale) fit the edge device.

## Risks and open questions
- Band boundaries need component frequencies (RPM, bearing geometry, gear teeth). Ask Wave whether this is known per machine.
- Faults that leak across bands (e.g. a bearing fault with broadband noise) may raise several scores; needs a cross-band rule.
- k models → k× memory and compute; check the edge budget (or share one backbone with per-band heads).
- Needs enough labelled normal variation (several machines or mics) for auxiliary classification; otherwise fall back to kNN on embeddings.
