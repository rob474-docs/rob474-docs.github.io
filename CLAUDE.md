# Writing style for this site

This is course documentation for ROB 474/574 (Uncrewed Aerial Systems). The
audience is upper-level undergraduate and graduate engineering students. Write
the way good engineering documentation and literature reads.

## Punctuation

**Never use em dashes.** Restructure the sentence instead. A full stop, a
colon, a comma, parentheses, or a connective such as "and", "because", "so" or
"which" almost always reads better. The same applies to commit messages and to
any file in this repository.

Use the en dash only for numeric ranges (0.5-2 m, Parts 6-8). Hyphenate
compound modifiers (closed-loop control, 3-position switch).

## Register

- Prefer declarative sentences. State the requirement, then the reason.
- Use the imperative for procedures: "Set `EKF2_OF_CTRL` to 1", not "you'll
  want to set".
- Keep the second person for instructions to the student. Avoid conversational
  filler, rhetorical questions, and asides addressed to the reader.
- Report measured results as data, with units and conditions. "Height held to
  within 5 cm over a 15 s hover" rather than "it held height really well".
- Observations from instructor testing are worth keeping because they tell
  students what to expect, but state them as findings, not anecdotes: "On the
  course airframe this settles in about 0.3 s with no overshoot."
- Define a term or parameter the first time it appears, then use it plainly.
- Do not oversell. No "powerful", "seamless", "simply", "just".

## Structure

- Numbered steps for procedures, tables for parameters and troubleshooting.
- Put the safety-critical constraint before the step it governs, not after.
- Cross-reference with Jekyll `{% link %}` tags so links survive renames.
