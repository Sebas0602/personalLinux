# The Command Line Murders: Detective's Log Assignment

## What you're doing

There's been a murder in Terminal City, and the only tools you get are the ones on the command line. You'll solve the Command Line Murder Mystery (clmystery), a puzzle written by Noah Veltman, using `cat`, `grep`, `head`, `tail`, `ls`, `cd`, `wc`, pipes, and whatever else you've picked up so far.

Solving the case is only part of the job. The bigger part is the **detective's log**: a record of every move you made, why you made it, and what happened.

## Why the log matters more than the answer

The name of the killer is on the internet. An AI chatbot can tell you in two seconds. So the name by itself proves nothing about what you learned, and it's worth very few points.

Think of a real detective. One who says "I just know it was him" gets laughed out of court. One who says "The receipt put him at the store at 9:40, the camera shows his car leaving at 9:52, and the witness saw that car on Elm Street" wins the case. The judge wants the chain of evidence. So do I.

The log is also where the learning actually happens. When you write down *why* you're about to run a command, you catch yourself guessing. When you write down what you *expected* to see, and then you see something else, that surprise is the moment your understanding of the tool changes. Skip the writing and you skip that moment.

## Setup

1. Open your terminal (Codespaces or the lab VM).

2. Make your prompt show the time. Paste this and press Enter:

   ```bash
   export PS1='[\t] \u@\h:\w\$ '
   ```

   Now every line of your prompt starts with a clock, like `[14:05:31]`. Your screenshots will carry timestamps without any extra effort. If you close the terminal and come back later, run this line again.

3. Get the mystery:

   ```bash
   git clone https://github.com/veltman/clmystery.git
   cd clmystery
   ```

4. Read the rules of the game. The mystery itself asks you **not** to open its case files in a text editor. The only files you may open in an editor like `nano` are `instructions`, `cheatsheet.md`, and the `hint` files. Everything else you investigate with commands. That rule is the whole point of the exercise, so please respect it.

## How the log works

Your log is a numbered list of **steps**. A step is one decision: you looked at something, you thought about it, and you ran a command because of it.

Every step has three parts.

**Part 1: The data.** A screenshot of the evidence you're looking at right now. This is usually the output from your previous command.

**Part 2: The reasoning.** In your own words, a few sentences:
- What do you notice in this data?
- What do you want to find out next?
- What do you **expect** your next command to show? Write the prediction down before you run it.

**Part 3: The command and result.** A screenshot showing the command you ran and what came back. Then one line: did the result match your prediction? If not, what was different?

Here's a nice shortcut. The result of step 5 is the data for step 6. So you don't need two copies of the same screenshot. For step 6, Part 1 can simply say "see result of step 5" and you go straight to your reasoning.

### A worked example: Step 1

**Data:** The README of the repository says to start by reading a file called `instructions`, and says not to open the case files in a text editor.

**Reasoning:** The README tells me where the story starts. Since I'm not supposed to use an editor, I'll print the file to the screen with `cat`. I expect some kind of story setup, and maybe a description of what files exist and how clues are marked.

**Command and result:** `cat instructions` *(screenshot here)*. Prediction check: yes, it's the setup of the case. It also told me something I didn't expect: *(whatever surprised you)*.

### Weak reasoning vs. strong reasoning

Weak: "I ran cat on the instructions."

That just repeats the command in English. It tells me what your fingers did, not what your head did.

Strong: "The README told me the case starts in `instructions` and that I shouldn't use an editor. So I'll print it with `cat`. I expect a story setup and maybe a list of the files."

This one names the evidence, explains the choice of tool, and makes a prediction I can check against the screenshot.

You don't need fancy language. Short, plain, honest sentences are perfect. Write the way you'd explain it to a friend sitting next to you. Spelling and grammar don't count. Your own voice does.

## What counts as a step

- Any command that changes what you know about the case gets its own step.
- Small navigation commands (`cd`, a quick `ls` to remember a filename) can be bundled into the next real step. Just mention them.
- **Wrong turns are steps too.** If you searched for something and got nothing, or followed a lead that went nowhere, log it with the same three parts. Then add one line about what you learned from the miss. Dead ends earn full credit. Real investigations are full of them, and a log with zero wrong turns is honestly less believable than one with five.

## Hints

The mystery comes with files named `hint1` through `hint8`. You may use them. If you open a hint, log it as a step: screenshot the hint, and in your reasoning explain what you were stuck on and what the hint changed in your thinking. Using hints costs you nothing. Hiding that you used them is a problem.

## Working with your group

Talk strategy with your group as much as you like. Explain commands to each other. Argue about suspects. That's encouraged.

What you can't do is share screenshots or reasoning text. Every screenshot in your log comes from **your own terminal**, and every sentence of reasoning comes from **your own head**.

After you've solved the case, find one person in your group whose path was different from yours. Write two or three sentences at the end of your log: where did your investigations split, and whose route was faster or cleaner?

## Checking your answer

When you think you know who did it, read the file called `solution` with `cat`. It explains how to check your answer from the command line. Take a screenshot of the check. That screenshot is your final step.

## Wrapping up

As the very last thing, save your command history to a file and include it with your submission:

```bash
history > my_history.txt
```

Open it and glance through. It's a nice feeling to see the whole case laid out in commands.

## What to submit

One PDF containing:

1. Your numbered log of steps, each with the three parts.
2. The final step showing your answer check.
3. Your two or three sentences comparing routes with a group member.
4. A short final reflection (three to five sentences): What command or trick did you learn during this case that you didn't know before? Which step was the hardest, and why?

Plus the file `my_history.txt`.

Submit both on Brightspace by **[DUE DATE]**.

## How it's graded

| Part | Points | What I'm looking for |
|---|---|---|
| Completeness of the log | 30 | Every real step has data, reasoning, and command with result |
| Quality of reasoning | 35 | Your reasoning connects the evidence to the command. Predictions are written before results. Surprises are noted. |
| Honest wrong turns and hints | 10 | Dead ends and hint use are logged, with what you learned from them |
| Group comparison | 10 | A real comparison of two different routes |
| Final reflection | 10 | Specific, not generic |
| Correct answer | 5 | The check in the `solution` file says you got it |
| **Total** | **100** | |

I may ask you in class to walk me through one or two of your steps out loud. If you did the work, that conversation will be easy and fun.

## A last word

Getting stuck is part of the game. When you're stuck, go back and reread your last few pieces of reasoning. Very often the next clue is sitting in a screenshot you already took, and you just haven't asked it the right question yet.

Good luck, detective.
