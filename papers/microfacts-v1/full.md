---
doc_id: microfacts-v1.full
paper: microfacts-v1
version: 2026-02-23
---

# Microfacts v1 (Tiny-agent sanity + lookup pack)

Indexing note: all positions are **0-based** unless otherwise stated.

## Section 01 — String sanity: strawberry

Q: How many 'r' letters are in the word "strawberry"?
A: 3.
Proof: s t r a w b e r r y → 'r' appears three times (0-based positions 2, 7, 8; 1-based positions 3, 8, 9).

## Section 02 — String sanity: mississippi

Q: How many 's' letters are in the word "mississippi"?
A: 4.
Proof: m i s s i s s i p p i → 's' appears four times (0-based positions 2, 3, 5, 6).

Q: How many 'i' letters are in the word "mississippi"?
A: 4.
Proof: m i s s i s s i p p i → 'i' appears four times (0-based positions 1, 4, 7, 10).

Q: How many 'p' letters are in the word "mississippi"?
A: 2.
Proof: m i s s i s s i p p i → 'p' appears twice (0-based positions 8, 9).

## Section 03 — String sanity: banana

Q: How many 'a' letters are in the word "banana"?
A: 3.
Proof: b a n a n a → 'a' appears three times (0-based positions 1, 3, 5).

Q: How many 'n' letters are in the word "banana"?
A: 2.
Proof: b a n a n a → 'n' appears twice (0-based positions 2, 4).

## Section 04 — String sanity: occurrence

Q: How many 'c' letters are in the word "occurrence"?
A: 3.
Proof: o c c u r r e n c e → 'c' appears three times (0-based positions 1, 2, 8).
Note: Models commonly answer 2. The third 'c' at position 8 (0-based) is the trap.

Q: How many 'r' letters are in the word "occurrence"?
A: 2.
Proof: o c c u r r e n c e → 'r' appears twice (0-based positions 4, 5).

## Section 05 — String sanity: accommodate

Q: How many 'c' letters are in the word "accommodate"?
A: 2.
Proof: a c c o m m o d a t e → 'c' appears twice (0-based positions 1, 2).

Q: How many 'm' letters are in the word "accommodate"?
A: 2.
Proof: a c c o m m o d a t e → 'm' appears twice (0-based positions 4, 5).
Note: "accommodate" has double-c AND double-m. Models frequently miss one of the pairs.

## Section 06 — String sanity: necessary

Q: How many 'c' letters are in the word "necessary"?
A: 1.
Proof: n e c e s s a r y → 'c' appears once (0-based position 2).

Q: How many 's' letters are in the word "necessary"?
A: 2.
Proof: n e c e s s a r y → 's' appears twice (0-based positions 4, 5).
Note: Mnemonic: "one collar, two socks" (1c, 2s).

## Section 07 — String sanity: embarrass

Q: How many 'r' letters are in the word "embarrass"?
A: 2.
Proof: e m b a r r a s s → 'r' appears twice (0-based positions 4, 5).

Q: How many 's' letters are in the word "embarrass"?
A: 2.
Proof: e m b a r r a s s → 's' appears twice (0-based positions 7, 8).
Note: "embarrass" has double-r AND double-s. Both pairs are commonly missed.

## Section 08 — String sanity: programming

Q: How many 'm' letters are in the word "programming"?
A: 2.
Proof: p r o g r a m m i n g → 'm' appears twice (0-based positions 6, 7).

Q: How many 'g' letters are in the word "programming"?
A: 2.
Proof: p r o g r a m m i n g → 'g' appears twice (0-based positions 3, 10).
Note: The two 'g's are far apart (0-based positions 3 and 10), making this a common miss.

## Section 09 — String sanity: millennium

Q: How many 'n' letters are in the word "millennium"?
A: 2.
Proof: m i l l e n n i u m → 'n' appears twice (0-based positions 5, 6).

Q: How many 'l' letters are in the word "millennium"?
A: 2.
Proof: m i l l e n n i u m → 'l' appears twice (0-based positions 2, 3).
Note: "millennium" contains 2×l, 2×n, and 2×m (m at 0 and 9). All are common traps.
