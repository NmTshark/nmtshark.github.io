---
author: NmToan
date: 2026-04-18T04:59:04.866Z
lastmod: 2026-04-18T13:39:20.763Z
title: Soulmate Squ1rrel Writeup
slug:  
featured: false
draft: false
tags:
  - Misc
  
description:  None
  
---
## Challenge overview
Describe:
the great church of scientology is, as always, in search of wife candidates for the Scientology Freedom Medal of Valor-winning actor tom cruise.

this used to be a manual process, but apparently these days you can automate anything with ai?

https://soulmate.squ1rrel.dev

At first glance, this challenge looks like a simple web app: enter a birthday and get a generated “soulmate”.
However, after reading the source code, the real goal becomes clear:

- the backend generates a face
- a celebrity classifier scores that face
- if the **Tom Cruise** score is at least `0.15`, the backend returns the flag

---

## Initial inspection

The most relevant files were:

- `backend/app.py`
- `models/inference.py`
- `frontend/static/js/app.js`
- `frontend/templates/index.html`

### Analysis

The frontend only shows the public flow of the app.
The real solve path comes from the backend and model code, because that is where the score is computed and the flag condition is implemented. 

---

## Finding the real success condition

In `backend/app.py`, the important routes are:

- `GET /generate-random`
- `POST /submit-u`
- `GET /health` 

The important detail is that the flag is only returned when the backend determines that the generated candidate has a high enough **Tom Cruise** score.
If the score is below the threshold, the server returns a rejection message.
If the score is above the threshold, it reads `flag.txt` and returns the flag in JSON. 

### Analysis

This shows that the challenge is not about guessing a birthday or exploiting a normal web bug.
The real task is to make the generated image score highly enough as Tom Cruise.

---

## Understanding how the score is computed

In `models/inference.py`, the code defines:

```python
TOM_SCORE_THRESHOLD = 0.15
```

The backend uses a celebrity classifier and extracts the probability for the label `"Tom Cruise"`.
That value is compared against the threshold. If it is at least `0.15`, the challenge is solved. 

So the problem becomes:

> Find an input that makes the generated face score at least `0.15` as Tom Cruise.

---

## Why the frontend is not enough

From the frontend code, the visible UI only uses `GET /generate-random`.
That endpoint derives a seed from a birthday and generates one random face. 
If we only use the frontend, we do not have much control.
We are basically hoping that some birthday produces a face that happens to score highly as Tom Cruise.

That is possible in theory, but it is not the intended way to solve the challenge.

---

## Discovering the intended solve path: `POST /submit-u`

The important backend endpoint is:

```http
POST /submit-u
```

This endpoint accepts a vector `u`, maps it through the PCA latent mapper, generates a face, scores the face with the celebrity classifier, and returns the result.
If the Tom Cruise score reaches the threshold, the flag is returned.

This is the key observation of the challenge.

Instead of controlling the model indirectly through a birthday, we can control it directly through a latent vector.
That changes the problem into a search problem:

> Find a control vector `u` that makes the Tom Cruise score exceed `0.15`.

---

## 6. Querying `/health` to understand the search space

Before searching, it helps to inspect the service configuration.

```bash
curl -sS https://soulmate.squ1rrel.dev/health
```

The health endpoint reveals that:

- the threshold is `0.15`
- the PCA mapper is loaded
- the control dimension is `8`
- each coordinate has lower and upper bounds. 

This means the optimization problem is:
- input: an 8-dimensional vector
- each dimension is bounded
- objective: maximize the Tom Cruise score

That is a small enough space that simple search methods are sufficient.

---

## 7. Verifying the oracle with a simple input

The first check is to send the zero vector.

### Script: `zero_test.py`

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

### Expected result

This should return a Tom Cruise score below the threshold.
In the original write-up, the zero vector gave a score around `0.0691`. 

This confirms that `/submit-u` can be used as a **scoring oracle**:

- we choose `u`
- the backend generates a face
- the backend tells us the Tom Cruise score

---

## 8. Measuring the effect of individual PCA dimensions

Before running a broader search, it is useful to test each PCA coordinate separately while keeping the other dimensions fixed at zero.

### Script: `axis_probe.py`

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


This step helps answer:

- can one coordinate solve the challenge by itself?
- which directions seem useful?
- do we need a combination of multiple coordinates?

The conclusion from the original solve was that no single coordinate alone was enough to cross the threshold, so a combination of dimensions was necessary. 

---

## 9. Random search for a good starting point

Once the search space is understood, the next step is to sample random vectors inside the valid bounds and keep the best one found so far.

### Script: `random_search.py`

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

The purpose of random search is not necessarily to solve the challenge immediately.
Its main purpose is to find a promising starting point in the search space.

In the original write-up, the best random candidate reached about:

```text
0.0942730382
```

with a vector close to:

```text
[-12.9805, 21.9629, 1.8578, 11.0075, 0.5764, 4.5666, -11.1215, 2.4167]
```

### Note

Random search can involve some luck.
Depending on the sampled vectors, it may find a strong candidate quickly, or it may take longer to reach a good score.

---

## 10. Refining the candidate with coordinate search

After obtaining a strong candidate from random search, the next step is to refine it.

The idea is:

1. keep 7 coordinates fixed
2. sweep the remaining coordinate across its valid range
3. keep the value that improves the score the most
4. repeat for the next coordinate

This is a simple coordinate search / hill-climbing method.

### Script: `coordinate_search.py`

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

# starting point from random search
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


This is the step that turns a good candidate into a successful one.

According to the original write-up:

- adjusting dimension 0 improved the score to about `0.140278`
- adjusting dimension 1 then improved it to about `0.151170`

That was enough to cross the threshold and recover the flag.

---

## 11. Submitting the final vector

Once the final vector is known, it can be submitted directly.

### Script: `final_submit.py`

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

### Expected result

The backend returns the flag when this vector is submitted. The flag is:

```text
squ1rrel{7h3_church_c0n6r47ul4735_mr5_cru153!!}
``` 

