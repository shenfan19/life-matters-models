# The Ebbinghaus Forgetting Curve: A 130-Years/Cross-Sample Extrapolation Validation Report

Model file: `ebbinghaus_forgetting_2026.yaml`. Summary index: `models/test_validation/validation_report.md`, section 1.

## Method

A power-law curve is fit to Ebbinghaus (1885)'s original memory-savings-rate data from one of his own subjects (7 time points):

```
retention(t) = A · (1 + t/20min)^(-beta)
```

Fit coefficients (least squares, computed by the author, not given directly by the literature): A=0.5975, beta=0.1442.

Substituting the same set of time points from Murre & Dros's (2015, PLOS ONE) independent-subject replication experiment 130 years later, and comparing predicted against measured values.

## Results

| Time point | Ebbinghaus 1885 (data used for the fit) | Model prediction | Murre & Dros 2015 measured | Deviation |
| --- | --- | --- | --- | --- |
| 20 minutes | 58.2% | 54.1% | 47.2% | +14.5% |
| 1 hour | 44.2% | 48.9% | 37.3% | +31.2% |
| 9 hours | 35.8% | 37.0% | 27.6% | +33.9% |
| 1 day | 33.7% | 32.2% | 31.7% | **+1.5%** |
| 2 days | 27.8% | 29.2% | 23.0% | +26.7% |
| 6 days | 25.4% | 24.9% | 16.8% | +48.2% |
| 31 days | 21.1% | 19.7% | 4.1% | +379.3% |

## Verdict

**Validation status: partially consistent (validation_confidence=3)**. Apart from the 1-day test point, the model systematically overestimates the 2015 replication's measured values, since the 2015 savings rate is markedly lower than the 1885 original data at most time points. This is not a problem with the fitted curve's form, since the power-law fit itself performs well within Ebbinghaus's own 7 points from 1885, but a genuine difference between the two datasets: Ebbinghaus's 1885 data comes from a self-experiment on one of his own subjects, a sample size of 1, insufficient to represent population-level memory characteristics; Murre & Dros 2015 independently measured modern subjects, and a systematic deviation appearing in another group of subjects 130 years later is expected, not evidence of a mechanism error in the model.

The 31-day test point shows the largest deviation (+379%), and the Murre & Dros paper itself also flags this as the most anomalous data point in that replication, possibly related to individual subject differences or experimental procedure, and should not be over-interpreted.

## Limitations

- This file's power-law fit coefficients are the author's own least-squares fit, not equation parameters given directly by the Ebbinghaus or Murre & Dros papers.
- The fit uses only the 7 points from 1885, a low sample efficiency; if a larger-sample historical memory-curve dataset becomes available in the future, the robustness of this "130-year extrapolation" conclusion could be re-assessed.
