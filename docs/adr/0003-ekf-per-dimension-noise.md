# ADR-0003: Per-dimension process/measurement noise in the geodetic EKF

**Status:** Accepted (Phase 3, 2026-08-23)

## Context

`fusion-svc`'s Extended Kalman Filter tracks a 6-state vector spanning three physically different units and scales: lat/lon (degrees, ~1e-5 scale changes matter), altitude (meters), and N/E/U velocity (m/s). A first implementation used one scalar process-noise value and one scalar measurement-noise value shared across every state dimension — the same shortcut `cv-inference-svc`'s DeepSORT Kalman filter (ADR-0002) uses safely, because its entire state is pixels, one unit, one scale.

## Decision

`EkfNoiseConfig` gives each state dimension its own variance (`posDeg2`, `altM2`, `velMps2`, ...) rather than one shared scalar. The Jacobian is recomputed per step to handle the lat/lon coupling nonlinearity, rather than using a relabeled linear KF.

## Consequences

- The filter actually learns velocity from repeated position innovations. The single-scalar version silently failed this: a variance tuned for the degree-scale position dimension was orders of magnitude too small for the m/s velocity dimension, so velocity states never accumulated enough covariance for the Kalman gain to update them — position converged, heading/speed stayed pinned near zero. This looked like a logic bug and was pure noise-tuning; caught by a real assertion on velocity convergence, not by inspection.
- Any future state addition to this EKF (e.g. angular rate, acceleration) needs its own noise term in `EkfNoiseConfig` — reusing an existing dimension's variance "because it's close enough" reintroduces this exact bug in a new dimension.
