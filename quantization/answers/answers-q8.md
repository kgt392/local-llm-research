# Answers — llama3.1:8b-instruct-q8_0 (~8.5 GB)

**Date:** 12 Aug 2026 · **Hardware:** MacBook Air M3, 16GB · **Ollama, --verbose**
**Speed:** ~11.2 tok/s across all questions — ~38% slower than q4/q2, consistent with ~1.7× more bytes to read per token (memory-bandwidth bound). Model still fits in RAM (no thrashing).

---

## Q1 — What causes the seasons on Earth? Explain in 4-5 sentences.
**eval rate: 11.23 tok/s**

The changing seasons on Earth are caused by the tilt of our planet's axis as it orbits around the Sun. At an angle of about 23.5 degrees, the North Pole is tilted towards or away from the Sun throughout the year, resulting in varying amounts of sunlight and warmth received at different latitudes. When the North Pole is tilted towards the Sun, the Northern Hemisphere receives more direct sunlight, leading to longer days and warmer temperatures during summer. Conversely, when it's tilted away, the hemisphere experiences shorter days and colder temperatures during winter. This tilt also affects the Southern Hemisphere, resulting in its own seasonal changes six months later.

**Scoring notes:** Correct and slightly richer than q4 (latitudes, six-month offset).

---

## Q2 — Shopkeeper pens problem (correct answer: profit ₹120)
**eval rate: 11.22 tok/s**

To find the shopkeeper's total profit or loss, we need to calculate the cost price (CP) and selling price (SP).

**Step 1: Calculate the total cost price**

Cost of buying 40 pens = Number of pens × Price per pen
= 40 pens × 12 rupees/pens
= 480 rupees

**Step 2: Calculate the revenue from selling 25 pens at 18 rupees each**

Revenue from 25 pens = Number of pens × Selling price per pen
= 25 pens × 18 rupees/pens
= 450 rupees

**Step 3: Calculate the revenue from selling the remaining pens (15 pens) at 10 rupees each**

Number of remaining pens = Total pens - Pens sold earlier
= 40 pens - 25 pens
= 15 pens

Revenue from 15 pens = Number of pens × Selling price per pen
= 15 pens × 10 rupees/pens
= 150 rupees

**Step 4: Calculate the total revenue**

Total revenue = Revenue from 25 pens + Revenue from 15 pens
= 450 rupees + 150 rupees
= 600 rupees

**Step 5: Find the profit or loss**

Profit/Loss = Total revenue - Total cost price
= 600 rupees - 480 rupees
= 120 rupees

Since the result is positive, there is a **profit of 120 rupees**.

**Scoring notes:** Perfect — clean structure, correct ₹120.

---

## Q3 — Python: second largest unique number + edge case
**eval rate: 11.18 tok/s**

Here's a simple function using Python:

```python
def second_largest(nums):
    """
    Returns the second largest unique number in a list.

    Args:
        nums (list): A list of integers.

    Returns:
        int: The second largest unique number, or None if it doesn't exist.
    """
    # Remove duplicates by converting to a set
    num_set = set(nums)

    # Handle edge case where there are less than 2 unique numbers
    if len(num_set) < 2:
        return None

    # Sort the set in descending order and return the second element
    sorted_nums = sorted(list(num_set), reverse=True)
    return sorted_nums[1]
```

Here's an explanation of how it works:

1. First, we remove duplicates by converting the list to a set.
2. We then check if there are at least 2 unique numbers in the set. If not, we return None.
3. Finally, we sort the set in descending order (largest to smallest) and return the second element.

Here's an example usage:

```python
print(second_largest([1, 2, 3, 4, 5]))  # Output: 4
print(second_largest([5, 5, 5, 5]))     # Output: None
```

Note that this function has a time complexity of O(n log n) due to the sorting operation. If you need a more efficient solution for large lists, consider using a data structure like a heap or a balanced binary search tree.

**Scoring notes:** Correct + docstring + correct example outputs + complexity note. Marginally more polished than q4.

---

## Q4 — Exactly 3 bullets, <12 words each, no "health"
**eval rate: 11.34 tok/s**

Here are three bullet points:

• Regular exercise boosts mood and reduces stress levels.
• Exercise improves sleep quality and duration significantly.
• Physical activity enhances cognitive function and memory recall.

**Scoring notes:** All constraints met — 3 bullets, 8/7/8 words, no "health".

---

## Q5 — Industrial Revolution 2-sentence summary
**eval rate: 11.22 tok/s**

Here is a 2-sentence summary:

The Industrial Revolution, sparked by innovations in textile production, steam power, and iron-making, transformed agrarian societies into industrial ones in Britain in the late 18th century. While it brought unprecedented economic growth and technological progress, it also led to harsh working conditions, social problems, and labor reform movements that had far-reaching impacts across Europe and America.

**Scoring notes:** Correct, both sides kept, 2 sentences.
