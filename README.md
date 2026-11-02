# ClearPath

![Python](https://img.shields.io/badge/python-3.9%2B-blue)
![Streamlit](https://img.shields.io/badge/built%20with-Streamlit-ff4b4b)
![Status](https://img.shields.io/badge/status-simulation%20only-orange)

**A Python simulation of a speed bump that flattens when it is confident it hears a siren, so an ambulance loses less time, and stays raised for everything else.**

> **Simulation only.** The sound options stand in for a siren detector. The app does not use a microphone, does not detect real sounds, and has not been tested on real roads. Every speed and timing value is an assumption, not a measurement.

## The simulation

Pick a sound and a vehicle, click **Run simulation**, and watch whether the bump goes flat in time. Here are three runs. The animations are rendered from the simulation's own drawing code.

**1. Siren heard early (80 m before the bump):** the siren is confirmed, the bump goes flat, and the ambulance loses much less time.

![Ambulance with an early siren](docs/demo_early_siren.gif)

**2. Siren heard too late (20 m before the bump):** the siren is detected, but the bump is still lowering when the ambulance arrives, so it slows anyway.

![Siren heard too late](docs/demo_late_siren.gif)

**3. Normal car, no siren:** nothing is confirmed, so the bump stays raised, which is today's behaviour.

![Normal car, bump stays raised](docs/demo_normal_car.gif)

### Run it yourself

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python3 -m streamlit run app.py
```

Then open http://localhost:8501, choose a sound, a vehicle and a siren distance, and click **Run simulation**.

---

## How it works

ClearPath is one Python file. Python runs the whole simulation: the detector rules, the bump, the vehicle and the scoring. Streamlit only shows what Python computes.

### 1. Architecture

```mermaid
flowchart TB
    subgraph UI["Streamlit page"]
        A1["Choose a sound: Siren, Car horn or Silence"]
        A2["Choose a vehicle: Ambulance or Normal car"]
        A3["Slider: how far before the bump the siren is first heard"]
        A4["Run simulation button"]
    end

    subgraph ENGINE["Python simulation engine"]
        B1["simulate(): steps the trip every 0.05 s"]
        B2["Simulated detector: one siren probability every 0.5 s"]
        B3["Bump controller: lowers on a confirmed siren, raises otherwise"]
        B4["Vehicle model: brakes for a raised bump, speeds up on flat road"]
        B5["evaluate(): correct, late, missed or false alarm, plus time saved"]
    end

    subgraph SHOW["What you see"]
        C1["scene_svg(): draws each frame of the road, bump and vehicle"]
        C2["Probability bar, bump state, counters and result message"]
    end

    A1 --> A4
    A2 --> A4
    A3 --> A4
    A4 --> B1
    B1 --> B2
    B2 --> B3
    B3 --> B4
    B4 --> B1
    B1 --> B5
    B1 --> C1
    B5 --> C2
    C1 --> C2
```

| Layer | Tool | Job |
|---|---|---|
| Logic | **Python** (`dataclasses`, `random`) | Detector rules, bump control, vehicle motion and scoring |
| Page | **Streamlit** | Buttons, slider, counters and messages |
| Drawing | **SVG built by Python** | Each animation frame is generated as text from the simulation numbers |

### 2. What happens at every time step

```mermaid
flowchart TD
    S["Start of a 0.05 s time step"]
    H["Work out what the microphone hears now"]
    R{"0.5 s since the last reading?"}
    P["New probability = 0.5 x old + 0.5 x target + small noise"]
    T{"Probability >= 0.8?"}
    K["Count one more reading in a row"]
    Z["Reset the count to zero"]
    C{"3 readings in a row?"}
    L["Siren confirmed: bump starts to lower"]
    M["Bump height moves toward flat or raised, taking 1.5 s"]
    V{"Is the bump flat?"}
    W["Vehicle brakes toward 4 m/s near the bump"]
    X["Vehicle keeps cruising at 16.7 m/s"]
    N["Move the vehicle and advance the clock"]
    E{"End of the road reached?"}
    F["evaluate() scores the run and shows the result"]

    S --> H
    H --> R
    R -- yes --> P
    R -- no --> M
    P --> T
    T -- yes --> K
    T -- no --> Z
    K --> C
    Z --> M
    C -- yes --> L
    C -- no --> M
    L --> M
    M --> V
    V -- no --> W
    V -- yes --> X
    W --> N
    X --> N
    N --> E
    E -- no --> S
    E -- yes --> F
```

The siren is **confirmed** only after 3 readings in a row at or above 0.8. The bump then needs 1.5 s to go flat. It starts rising again 3 s after the siren stops.

### 3. The bump's states

```mermaid
stateDiagram-v2
    [*] --> Raised
    Raised --> Lowering: siren confirmed
    Lowering --> Flat: bump height reaches zero after 1.5 s
    Flat --> Rising: no siren readings for 3 s
    Lowering --> Rising: confirmation lost
    Rising --> Raised: bump height reaches full
    Rising --> Lowering: siren confirmed again
```

**Fail-safe rule:** the bump starts raised and only lowers after a confirmed siren. When unsure, nothing changes, so the worst case is today's behaviour.

### 4. Why timing decides everything

The bump can only help if it is flat when the vehicle arrives. That needs a warning of

```
warning needed = speed x (time to confirm the siren + 1.5 s to flatten)
```

The app measures the confirmation time in each run and shows the warning distance it implies. A siren heard too close to the bump is detected correctly but still **too late**.

### 5. What each run is scored as

| Vehicle | Bump when the vehicle arrives | Result |
|---|---|---|
| Ambulance | Flat | Correct |
| Ambulance | Siren confirmed, but still lowering | Late |
| Ambulance | Siren never confirmed | Missed |
| Normal car | Raised | Correct |
| Normal car | Lowered | False alarm |

The time saved is the trip time of a run with the bump always raised, minus the trip time of this run.

### 6. Functions in `app.py`

| Function | Job |
|---|---|
| `simulate()` | Steps the trip forward: detector readings, bump height and vehicle speed |
| `evaluate()` | Decides the outcome at the bump and the time saved against a raised bump |
| `scene_svg()` | Draws the road, bump and vehicle from the numbers |
| `main()` | Lays out the Streamlit page |

---

## What the simulation shows

| Scenario | What happens |
|---|---|
| Siren heard early | The siren is confirmed, the bump goes flat, and the ambulance loses much less time |
| Siren heard too late | The bump is still lowering when the ambulance arrives, so it slows anyway |
| Normal car, no siren | The bump stays raised, which is today's behaviour |
| Normal car while a siren plays elsewhere | **False alarm**: the bump lowers when it should not |

### Example results (default settings)

These are outputs of the simulation under the assumptions below, not real-world measurements.

| Siren first heard | Outcome | Time saved vs a raised bump |
|---|---|---|
| 100 m before the bump | Correct | 5.8 s |
| 80 m | Correct | 5.1 s |
| 60 m | Correct | 4.1 s |
| 40 m | Correct | 1.9 s |
| 20 m | Late | 0.4 s |

With the default assumptions, a raised bump costs the ambulance about 6 seconds.

## The problem

Speed bumps slow ambulances down. A peer-reviewed measurement study of ambulance delay at speed humps ([PMC9922345](https://pmc.ncbi.nlm.nih.gov/articles/PMC9922345/)) reported roughly 8 to 10 seconds lost at a single hump (Turkish roads, so local results will differ). ClearPath asks a narrower question:

> If a bump could lower itself on a confirmed siren, how much warning does it need, and what happens when detection is late or wrong?

## Assumptions (all of these are editable in `Params`)

| Parameter | Default | Meaning |
|---|---|---|
| `cruise_mps` | 16.7 | Ambulance speed, about 60 km/h |
| `cross_mps` | 4.0 | Speed over a raised bump |
| `decel_mps2` / `accel_mps2` | 2.5 / 1.5 | Braking and speeding up |
| `threshold` | 0.8 | Probability needed to count a reading as "siren" |
| `confirm_readings` | 3 | Readings in a row needed to confirm |
| `retract_time_s` | 1.5 | Time for the bump to go flat |
| `hold_time_s` | 3.0 | Bump rises this long after the siren stops |

The detector's probability comes from a simple formula (siren 0.93, car horn 0.22, silence 0.03, smoothed with a little noise). It is **not a trained model**.

## Limitations

- It is a simulation. Nothing here was tested on a real road or with real ambulances.
- No microphone input and no trained detector. The probability is generated by a formula.
- The assumed speeds and timings are placeholders until replaced with measured data.
- Pedestrians crossing at the bump, traffic queues ahead, multiple vehicles and spoofed sirens are not modelled.
- Session counters in the app are not a measured accuracy.
- Any real installation would need safety testing and certification.

## Related work

The idea of a retractable or siren-triggered speed bump is not new. Examples include a [patent for a retractable speed bump controlled from emergency vehicles](https://patents.google.com/patent/US5509753), a [patent describing audio siren detection that lowers the bump](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/9951487), and a [research paper on smart speed bumps with LoRa](https://arxiv.org/pdf/2307.00433). ClearPath does not claim to invent the concept. It is a small, transparent model for exploring timing and failure cases.

## Next steps

- Record real siren, horn and traffic sounds and train a detector with a leak-free evaluation
- Replace the assumed values with a measured speed trace over a real bump
- Add a traffic-queue scenario and a pedestrian-hold rule

## Project files

```
app.py            the whole simulation and page
requirements.txt  Python dependencies
docs/             animated previews used in this README
```
