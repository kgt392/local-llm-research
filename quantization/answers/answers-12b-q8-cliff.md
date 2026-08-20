# The Cliff Specimen — gemma3:12b-it-q8_0 (13 GB on 16 GB RAM)

**Date:** 10 Aug 2026 · **Hardware:** MacBook Air M3, 16GB · **Ollama, --verbose**

## The numbers (Part 1 of the study: what happens when the model does NOT fit)
```
eval rate:          0.17 tokens/s   (vs ~11 tok/s for the same model at q4)
prompt eval rate:   0.44 tokens/s
1067 tokens took:   ~1h44m
Memory Used:        15.4-15.6 / 16 GB (maxed)
Wired:              ~10.3-10.5 GB
Swap:               ~3.25-3.4 GB
Compressed:         ~3-4.6 GB
Memory Pressure:    RED (thrashing)
CPU (llama-server): ~33% — mostly WAITING on SSD, not computing
```

**Interpretation:** the 13 GB model + macOS exceeds 16 GB physical RAM. For every token, several GB of weights must be paged in from swap (SSD) — the machine thrashes. q8 is only 1.6× larger than q4 but ran ~65× slower.

> **The cliff law: speed is not proportional to model size. Fits in RAM → fast. Doesn't fit → falls off a cliff. The marginal GB beyond RAM costs ~100×.**

Only one question was asked — the result was unambiguous, so the run was stopped (no need to torture the machine for 4 more answers).

---

## Q — Explain how a refrigerator works.
*(Answer quality itself was FINE — the problem was purely speed. Quality doesn't degrade when a model swaps; it just becomes unusably slow.)*

Okay, let's break down how a refrigerator works. It seems simple (keep things cold!), but there's some clever physics and engineering involved. Here's an explanation, starting with the basic idea and then going into more detail about the components:

**The Core Concept: Moving Heat, Not Making Cold**

The most important thing to understand is that refrigerators *don't* "make" cold. They **move heat** from inside the fridge to outside. Think of it like a pump moving water – it doesn't create water, it just relocates it. Coldness is simply the absence of heat. So, by removing heat from the refrigerator's interior, we make it feel colder.

**The Key Components & How They Work Together (the Refrigeration Cycle):**

A refrigerator uses a cycle involving four main components and a special fluid called a **refrigerant**. This refrigerant cycles through different states (liquid, gas) to absorb and release heat:

1. **Compressor:** takes the refrigerant in its gaseous state (low pressure, low temperature) and compresses it into a high-pressure, high-temperature gas. Compressing the gas significantly increases its energy level, which allows it to release heat effectively later on.

2. **Condenser Coils:** black, finned coils on the *outside* of the refrigerator. The hot, high-pressure refrigerant gas flows through these coils and releases its heat into the room air, condensing back into a high-pressure liquid. The fins increase surface area for more efficient heat transfer.

3. **Expansion Valve (or Capillary Tube):** a small, narrow tube/valve that dramatically reduces the pressure of the liquid refrigerant. When the pressure drops suddenly, some of the liquid instantly flashes into vapor without adding heat, causing a huge temperature drop.

4. **Evaporator Coils:** the coils *inside* the refrigerator. The very cold, low-pressure refrigerant flows through them; heat transfers from the warmer interior air to the colder refrigerant, which boils into a low-pressure gas. This is what chills the fridge's contents.

**The Cycle Repeats:** the low-pressure gas flows back to the compressor and the loop continues — constantly removing heat from inside the fridge.

Compressor → Condenser Coils → Expansion Valve → Evaporator Coils → Compressor…

**Additional points:** the thermostat switches the compressor on/off to hold the set temperature; frost-free models periodically melt frost with a heating element; heavy insulation minimizes heat entering the compartment.

**In short:** A refrigerator is a closed system that uses the properties of a refrigerant to move heat from inside the fridge to outside, keeping your food cold!
