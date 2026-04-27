---
author: NmToan
date: 2026-04-18T04:59:04.866Z
lastmod: 2026-04-27T00:00:00.000Z
title: Soulmate Squ1rrel Writeup
slug:
featured: false
draft: false
tags:
  - Misc
description: Write-up of the latent-vector optimization used to push the Tom Cruise score past the flag threshold
---

# Soulmate Squ1rrel Write-up

## Challenge Overview

The challenge presents a web app with the following premise:

```text
the great church of scientology is, as always, in search of wife candidates for the Scientology Freedom Medal of Valor-winning actor tom cruise.

this used to be a manual process, but apparently these days you can automate anything with ai?
```

Link:

- `https://soulmate.squ1rrel.dev`

At first glance, it looks like a simple app that takes a birthday and generates a "soulmate." But after reading the source code, the actual objective becomes clear:

- the backend generates a face,
- a celebrity classifier scores that face,
- if the **Tom Cruise** score reaches at least `0.15`, the backend returns the flag.

So this is not really a guessing challenge. It is an optimization problem over the model input.

## Solve Strategy

The entire approach can be summarized in four steps:

1. inspect the backend and find the real flag condition,
2. identify the endpoint that allows direct latent control,
3. use that endpoint as a scoring oracle,
4. optimize the vector `u` until `tom_score >= 0.15`.

## 1. Find the Real Success Condition

The most relevant files are:

- `backend/app.py`
- `models/inference.py`
- `frontend/static/js/app.js`
- `frontend/templates/index.html`

The frontend only exposes the public flow. The real solve path lives in the backend and model code.

In `models/inference.py`, the critical line is:

```python
TOM_SCORE_THRESHOLD = 0.15
```

The backend extracts the probability for the `"Tom Cruise"` label and compares it against that threshold. If the score is below `0.15`, the server returns a rejection. If it reaches the threshold, it reads `flag.txt` and returns the flag as JSON.

So the real problem is:

> Find an input that makes the generated face score at least `0.15` as Tom Cruise.

## 2. Why the Frontend Is Not the Right Path

From the UI, the visible route is:

- `GET /generate-random`

That endpoint derives a seed from a birthday and generates one face. If we only use that flow, we have almost no direct control and are basically hoping that some birthday happens to produce a face with a high Tom Cruise score.

That might work by chance, but it is not the clean or intended route.

## 3. The Important Endpoint: `POST /submit-u`

The key backend endpoint is:

```http
POST /submit-u
```

This endpoint accepts a vector `u`, maps it through the PCA latent mapper, generates a face, scores it with the celebrity classifier, and returns the result.

That changes the problem completely:

- instead of indirect control through a birthday,
- we get direct control over the latent vector.

At that point, the challenge becomes an 8-dimensional search problem:

> Find a vector `u` that maximizes the `"Tom Cruise"` score.

## 4. Query `/health` to Understand the Search Space

Before optimizing, I checked the service configuration:

```bash
curl -sS https://soulmate.squ1rrel.dev/health
```

The response reveals useful information:

- the threshold is `0.15`,
- the PCA mapper is loaded,
- `control_dim = 8`,
- every coordinate has lower and upper bounds.

So the optimization problem is:

- input: an 8-dimensional vector,
- each dimension is bounded,
- objective: maximize `tom_score`.

This is small enough for simple random search followed by coordinate refinement.

## 5. Verify the Oracle with the Zero Vector

The first sanity check is to submit the all-zero vector:

```python
import requests
import json

URL = "https://soulmate.squ1rrel.dev/submit-u"

u = [0.0] * 8
r = requests.post(
    URL,
    json={"u": u, "include_image": False},
    timeout=30,
)

print(json.dumps(r.json(), indent=2))
```

In the original solve, this returned a Tom Cruise score around `0.0691`, which is below the threshold.

That confirms that `/submit-u` works as a scoring oracle:

1. choose `u`,
2. let the backend generate a face,
3. read back the Tom Cruise score.

## 6. Probe Each Dimension Individually

Before a broader search, I tested each PCA coordinate independently while holding the others at zero:

```python
import requests
import json

BASE = "https://soulmate.squ1rrel.dev"

health = requests.get(BASE + "/health", timeout=30).json()

dim = health["control_dim"]
u_lower = health["u_lower"]
u_upper = health["u_upper"]

def query(u):
    r = requests.post(
        BASE + "/submit-u",
        json={"u": u, "include_image": False},
        timeout=30,
    )
    return r.json()

base = [0.0] * dim

for i in range(dim):
    for val in (u_lower[i], u_upper[i]):
        u = base[:]
        u[i] = val
        resp = query(u)
        print(f"dim={i}, val={val}")
        print(json.dumps(resp, indent=2))
```

This helps answer:

1. can a single dimension solve the challenge,
2. which directions are promising,
3. whether a combination of coordinates is required.

The conclusion was that no single coordinate was sufficient, so the solve required a multi-dimensional combination.

## 7. Random Search for a Good Starting Point

Once the search space was understood, I sampled random vectors within the allowed bounds and kept track of the best result:

```python
import random
import requests

BASE = "https://soulmate.squ1rrel.dev"

health = requests.get(BASE + "/health", timeout=30).json()

dim = health["control_dim"]
u_lower = health["u_lower"]
u_upper = health["u_upper"]
threshold = health["tom_score_threshold"]

def query(u):
    r = requests.post(
        BASE + "/submit-u",
        json={"u": u, "include_image": False},
        timeout=60,
    )
    return r.json()

best_u = [0.0] * dim
best_resp = query(best_u)
best_score = best_resp["tom_score"]

print("[*] start score:", best_score)
print("[*] start vector:", best_u)

for i in range(500):
    u = [random.uniform(u_lower[j], u_upper[j]) for j in range(dim)]
    resp = query(u)
    score = resp["tom_score"]

    if score > best_score:
        best_score = score
        best_u = u[:]
        print(f"[+] new best score: {best_score:.10f}")
        print(f"    vector: {best_u}")

    if score >= threshold:
        print("[+] threshold reached")
        print(resp)
        break
```

In the original run, random search did not solve the challenge immediately, but it found a strong candidate with a score around:

```text
0.0942730382
```

Vector:

```text
[-12.9805, 21.9629, 1.8578, 11.0075, 0.5764, 4.5666, -11.1215, 2.4167]
```

That was good enough to use as a refinement starting point.

## 8. Coordinate Search to Cross the Threshold

From that candidate, I switched to a simple coordinate-wise hill-climbing process:

1. keep 7 coordinates fixed,
2. sweep the remaining coordinate across its range,
3. keep the value that improves the score the most,
4. repeat over all dimensions for multiple rounds.

Script:

```python
import numpy as np
import requests

BASE = "https://soulmate.squ1rrel.dev"

health = requests.get(BASE + "/health", timeout=30).json()

dim = health["control_dim"]
u_lower = health["u_lower"]
u_upper = health["u_upper"]
threshold = health["tom_score_threshold"]

def query(u):
    r = requests.post(
        BASE + "/submit-u",
        json={"u": u, "include_image": False},
        timeout=60,
    )
    return r.json()

u = [-12.9805, 21.9629, 1.8578, 11.0075, 0.5764, 4.5666, -11.1215, 2.4167]

resp = query(u)
best_score = resp["tom_score"]

print("[*] initial score:", best_score)
print("[*] initial vector:", u)

for round_id in range(3):
    improved = False

    for i in range(dim):
        local_best_score = best_score
        local_best_val = u[i]

        for val in np.linspace(u_lower[i], u_upper[i], 40):
            cand = u[:]
            cand[i] = float(val)

            resp = query(cand)
            score = resp["tom_score"]

            if score > local_best_score:
                local_best_score = score
                local_best_val = float(val)

            if score >= threshold:
                print("[+] threshold reached")
                print("[+] winning vector:", cand)
                print(resp)
                raise SystemExit

        if local_best_score > best_score:
            u[i] = local_best_val
            best_score = local_best_score
            improved = True
            print(f"[+] improved dim {i}: score={best_score:.10f}")
            print(f"    vector={u}")

    if not improved:
        print("[*] no further improvement")
        break
```

In this solve:

- tuning dimension 0 improved the score to about `0.140278`,
- tuning dimension 1 pushed it to about `0.151170`.

That was enough to cross the `0.15` threshold.

## 9. Submit the Final Vector

Once the winning vector is known, it can be submitted directly:

```python
import requests
import json

URL = "https://soulmate.squ1rrel.dev/submit-u"

u = [-4.7957, 18.2934, 1.8578, 11.0075, 0.5764, 4.5666, -11.1215, 2.4167]

r = requests.post(
    URL,
    json={"u": u, "include_image": False},
    timeout=30,
)

print(json.dumps(r.json(), indent=2))
```

The backend returns:

```text
squ1rrel{7h3_church_c0n6r47ul4735_mr5_cru153!!}
```

## Conclusion

What makes this challenge fun is that it looks like a simple web input problem, but the real task is latent-space optimization:

1. the backend gates the flag on the Tom Cruise score,
2. `/submit-u` exposes direct latent control,
3. `/health` reveals the full bounded search space,
4. random search finds a promising seed,
5. coordinate search pushes it over the threshold.

Final flag:

```text
squ1rrel{7h3_church_c0n6r47ul4735_mr5_cru153!!}
```
