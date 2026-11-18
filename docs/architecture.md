# BEMS: Architecture

*One optimisation platform. Whatever the installation looks like.*

[← Back to BEMS](../README.md)

The [README](../README.md) describes what I'm trying to build: an energy management system that adapts to its installation instead of making the installation adapt to it. That raises a fairly obvious question: how do you design one optimiser for a house, a warehouse and potentially a fleet of electric trucks?

The answer isn't to write a separate optimiser for every battery or charger. **Separate what the equipment is, what we expect to happen, what we'd like to achieve and what the hardware can safely do.**

Our home is where BEMS moved from an optimisation idea to a working physical controller. It now combines forecasts, joint battery and EV planning, long-term energy valuation and an independent regulator to control real equipment. **BEMS Next takes that experience and turns it into a modular platform**, built around versioned site definitions, interchangeable resource models and a separate safety process.

## The installation is a model, not a collection of brand names

A battery, EV charger, heat pump and industrial BESS each have different internal dynamics. But the optimiser doesn't need to know their manufacturers. It needs contracts describing their capabilities, energy flows, limits, states and control interfaces.

A site also needs an explicit electrical topology, economic model and set of objectives. This is important: adding another charger is not just adding another button. It might share a phase, a fuse, a transformer or a contractual import limit with everything already installed.

```mermaid
flowchart TB
    Site["Site definition<br/>Resources, topology, tariffs, goals"] --> Models["Physical & economic models"]
    Devices["Device measurements"] --> Data["Qualified data"]
    Context["Prices, weather, operational schedules"] --> Data
    Data --> Forecast["Forecasts & uncertainty"]
    Data --> Planning["Planning"]
    Forecast --> Planning
    Models --> Planning
    Planning -->|"Targets"| Safety["Regulator & safety"]
    Data -->|"Live measurements"| Safety
    Safety -->|"Commands"| Devices
    Devices -->|"Feedback"| Data
    classDef plan fill:#e6efff,stroke:#5485cb,color:#17457c
    classDef safe fill:#e6f5eb,stroke:#3f9b66,color:#1b5635
    class Planning plan
    class Safety safe
```

BEMS Next already implements versioned site manifests, capability-based drivers and composable models for batteries, EV charging and thermal storage. Its offline tests cover very different installations, including multiple batteries and chargers, an EV-only site and a battery with thermal storage. **The optimiser is becoming installation-driven rather than device-specific.**

## Forecasting: the best prediction might not have the lowest error

Forecasting is where a seemingly small modelling choice can change an expensive physical decision.

It's tempting to choose the model with the lowest mean absolute error (MAE). After all, a forecast that is wrong by fewer watts should make the controller smarter. Except **not every watt of error matters equally**.

Imagine overestimating solar production by 500 W on a bright summer day when the battery is full and the house is already exporting electricity. It may barely affect the decision. The same 500 W error on a dark winter day, when nearly every available watt could replace expensive grid imports, may change whether the optimiser charges the battery or saves its remaining energy. Timing matters too: an error during an expensive peak hour can matter much more than the same error at 3 AM.

A fixed rule saying “winter matters more” is still an approximation. The real consequence depends on the battery's state, available flexibility, future prices, deadlines and whether the error changes a binding constraint.

### Why ordinary error metrics can mislead

MAE weights every evaluated watt of error equally. Summer, with much more solar production and more opportunities for large errors, can dominate the average even when winter decisions are economically more sensitive.

Normalised MAE (**nMAE**) doesn't automatically solve that. It depends on what we normalise by:

- Normalising by installed PV capacity gives a comparable percentage of rated power, but may still hide how consequential scarce winter energy is.
- Normalising by actual production can make errors explode on very low-production days, and becomes ill-defined at zero.
- Weighting by electricity price or estimated energy scarcity gets closer to the objective, but neither captures the entire downstream decision problem.

Household load is even harder. Cooking, heating, hot water and other loads don't reliably follow a smooth curve. **A good total-load forecast is not necessarily a good phase-load forecast.** A 6 kW load may be harmless when distributed across three phases, but exceed the limit of one phase if concentrated there. Some installations are constrained by a particular phase or branch, not by their total kW. Missing electrical topology cannot be fixed by simply training a larger model.

### The loss-function trap

The obvious next idea is to train a forecast on economic loss rather than prediction error. We've tried versions of this with solar forecasts, including price-weighted and asymmetric objectives. That exposed a different problem: **a forecast can look economically better because it compensates for mistakes elsewhere in the system.**

In one limited autumn battery-planning replay, an intentionally false forecast of **zero solar production** actually produced *lower simulated costs* than several more realistic forecasts. In a summer comparison, it was substantially worse. The false forecast had not discovered anything about the weather; it was making the optimiser behave more conservatively under the particular planning model and demand-charge assumptions.

A separate price-weighted training experiment improved its chosen price-weighted error measure, but made some peak-tariff error measures and solar overestimation worse. Improving one surrogate objective had simply moved the error somewhere else.

The improvement potential is measurable. In a retrospective study using archived GFS forecasts, adding a LightGBM residual to the solar physics model reduced scarcity-weighted absolute error from **0.351 to 0.246 kW, about 30%**. But that result is only half of the evaluation. The other half is whether the forecasts lead to better decisions when passed through the planner.

That is why defining the right loss function is difficult. If we train the prediction model to work around a particular MPC weakness, a later improvement to the planner could make the supposedly better forecast counterproductive. The forecast should describe a credible future; **the planner should decide how cautious to be about that future**.

```mermaid
flowchart TB
    Reality["Historical outcomes + issued forecasts"] --> Accuracy["Prediction quality<br/>MAE, bias, calibration, nMAE"]
    Reality --> Replay["Causal MPC replay<br/>Same constraints and initial state"]
    Accuracy --> Compare["Model selection"]
    Replay --> Compare
    Compare --> Decision["Does it improve decisions<br/>without distorting the forecast?"]
    classDef focus fill:#fff3dc,stroke:#d9a54a,color:#6a4916
    class Replay,Decision focus
```

So I don't want to optimise one metric and declare victory. Forecasts should be checked for accuracy, systematic bias, calibration, different seasons and horizons, and finally their effect on **out-of-sample decisions using the actual planner**. To make that comparison meaningful, a replay must use only the data and forecast versions available at each decision time. Otherwise, hindsight is being disguised as a forecast.

This is why BEMS separates physical solar models, statistical corrections, short-term measurement-based adjustments and scenario generation. They solve different problems. BEMS Next carries that separation into versioned, causal forecast-input contracts, so the optimiser can be tested with exactly the information available at each decision.

## Why two planners, and why stochastic optimisation?

A short-term optimiser can make the perfect decision for the next few hours and still get the week completely wrong.

Suppose electricity is cheap tonight. Filling the battery looks sensible. But if tomorrow's solar production could charge it instead, buying electricity tonight may be unnecessary. That solar is not free: using it to charge the battery means giving up potential export revenue. The right decision depends on purchase and export prices, conversion losses, available capacity and future needs. Or perhaps the car must leave early and tomorrow is cloudy. **Energy left in storage has a future value**, and that value changes with the expected future and the current state.

That's why BEMS separates the long horizon from the decisions it actually executes. The household long-term planner looks up to **168 hours ahead** and provides a state-dependent continuation value, `J`. Short-term model predictive control (MPC) uses that value alongside current measurements, near-term forecasts and user requirements. It doesn't have to solve the entire week again every time something changes.

```mermaid
flowchart TB
    Future["Possible futures<br/>Prices, load, solar, requirements"] --> Long["Long-term planner<br/>Future value J"]
    Future --> MPC["Short-term stochastic MPC"]
    Long -->|"Value of remaining energy"| MPC
    State["Current measured state"] --> MPC
    MPC -->|"Next control target"| Reg["Regulator"]
    Reg --> Hardware["Physical equipment"]
    Hardware -->|"New measurements"| State
    classDef plan fill:#e7edff,stroke:#7288d0,color:#294276
    class Long,MPC plan
```

The future isn't one guaranteed trajectory. BEMS therefore uses **stochastic optimisation**: possible outcomes for load, solar and prices, with decisions evaluated across them. The crucial rule is *non-anticipativity*: scenarios that are indistinguishable today must share today's decision. Future actions may differ only after the information that distinguishes those futures actually becomes available.

Each scenario has its own physical energy trajectory and constraints. Expected cost can be combined with downside-risk measures such as CVaR. Hard requirements, such as a vehicle needing sufficient charge at departure, remain constraints that the optimiser cannot sacrifice for a lower average bill.

These are working algorithms, not just proposed components. The household system combines joint stochastic MPC with adaptive future-state valuation. BEMS Next extends causal scenario planning and stored-state valuation across different resource models in its isolated runtime.

## The regulator doesn't trust the planner blindly

The planner makes an economic proposal. The regulator's priorities are simpler: **1. safety and control authority; 2. tracking the requested target**.

It reads live physical data independently, checks electrical limits, freshness, ownership and command validity, and only then controls the equipment. A command acknowledgement is not the same as a changed setting, and neither guarantees that physical power has already responded. The household controller has encountered exactly these timing problems.

Keeping this separate means a slow, failed or numerically unstable optimiser never gains physical authority. The real household controller already enforces this boundary. BEMS Next makes it an explicit process boundary too, with separate planning and safety processes, ownership fencing and failure tests in its simulated control loop.

## Agent-ready does not mean LLM-controlled

The idea is to let agents **build and maintain the system**, not decide exactly when the battery should charge.

For a new installation, an agent could inspect the equipment and its documentation, write a driver, connect unfamiliar data sources, implement a resource model or adapt the tariff logic. Ideally, the user shouldn't have to write that integration themselves. The generated code would still need unit checks, contract tests, simulation, replay and a controlled release before it can affect physical equipment.

The other job is dealing with major failures. In a system that controls real hardware continuously, something will eventually behave in a way nobody anticipated. Sensors drift, firmware changes, commands don't have the expected effect, and assumptions that worked yesterday stop being true. Rather than expecting every exception to have been programmed in advance, I'd like an agent to investigate unusual failures, identify the cause and help get the system back to normal. That might mean correcting an adapter, adjusting a validated model or invoking a specifically authorised recovery procedure.

**The immediate response must still come from ordinary control software.** The regulator needs to detect unsafe or invalid conditions and hold, stop or fall back without waiting for an LLM. An agent can work on the wider problem, but an untested code change must not become a live physical command.

```mermaid
flowchart TB
    Setup["New equipment or major fault"] --> Agent["Engineering agent<br/>Writes integrations and investigates failures"]
    Agent --> Checks["Tests, replay, validation"]
    Checks --> Release["Controlled release"]
    Release --> Control["Forecasting + MPC"]
    Control --> Safety["Independent regulator + safety"]
    Safety --> Hardware["Physical hardware"]
    Hardware -->|"Measurements"| Control
    Hardware -.->|"Fault evidence"| Agent
    classDef software fill:#e7efff,stroke:#5485cb,color:#17457c
    classDef safety fill:#e6f5eb,stroke:#3f9b66,color:#1b5635
    class Agent,Checks,Release software
    class Safety safety
```

A language model deciding the exact power setpoint every few seconds would be a terrible control system: slow, unpredictable and difficult to verify. Optimisation and feedback control should remain numerical, deterministic and bounded by independent safety rules. **If the agent is unavailable, physical control must remain safe.**

The goal is to make BEMS **agent-maintainable and adaptable**, while the numerical optimiser and physical safety loop remain independently dependable.

## Built today, designed to go further

**The operating system:** BEMS already controls real household equipment with solar, load and price forecasts, joint battery and EV planning, long-term valuation and a separate physical regulator. It also has an independent archive with Parquet and DuckDB for replay, research and evaluation.

**The modular foundation:** BEMS Next has working site manifests, composable battery, EV and thermal models, meter-topology checks, scenario optimisation, stored-state valuation and an independent simulated safety process. These components have been exercised in multiple offline installation configurations.

**The next step:** Connect this generalised architecture to verified real-device integrations and production deployment. The household implementation is operational; Next is the engineering path to making that capability reusable across installations.

BEMS started with one electricity bill. The architecture is being built for a much larger problem.

---

*Architecture and research summary based on BEMS and BEMS Next source and studies reviewed on 18 November 2026. The diagrams describe logical responsibilities, not physical wiring or completed universal hardware support.*