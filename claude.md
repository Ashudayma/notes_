# CLAUDE.md — Technical Interview Notes Generation

## Purpose

Convert video transcripts into **high-quality technical interview notes**. The final notes must be useful for learning, revision, interview preparation, and explaining concepts clearly in an interview.

The transcript is only the **source material**. Do not blindly copy it. Understand the technical content, remove noise, verify concepts, correct inaccuracies, and produce structured notes.

---

## 1. Remove Unnecessary Transcript Content

Ignore content that does not contribute to technical understanding, such as:

* Greetings, introductions, and outro messages
* Repeated explanations
* Filler words and casual conversation
* Personal stories unless they provide an important technical insight
* Marketing/promotional content
* Video/channel-specific commentary
* Unnecessary jokes or unrelated discussions
* Repeated examples that add no new information

Keep explanations, examples, comparisons, interview tips, practical insights, and technically relevant reasoning.

**Do not shorten the technical content just for the sake of being concise.** Remove noise, not important knowledge.

---

## 2. Make Notes Interview-Specific

The notes should prioritize what is useful in a **technical interview**.

For every important concept, cover relevant points such as:

* Definition — What is it?
* Why is it needed?
* How does it work?
* Internal working / architecture when relevant
* Important components
* Syntax or code examples when applicable
* Real-world use cases
* Advantages and disadvantages
* Limitations
* Common mistakes
* Differences from related concepts
* When to use and when not to use
* Performance / complexity considerations
* Important interview questions
* Follow-up questions an interviewer may ask

Focus especially on **WHY, HOW, and WHEN**, not just WHAT.

---

## 3. Correct Technical Inaccuracies

Never preserve an incorrect statement simply because it appeared in the transcript.

If the transcript contains:

* Incorrect technical information
* Outdated terminology
* Misleading explanations
* Incorrect code
* Wrong API/method names
* Incorrect comparisons
* Oversimplifications that could cause misunderstanding

**Correct it in the notes.**

Use your technical knowledge to validate the content. If something depends on a specific version, framework, library, or language version, clearly mention the relevant version.

Do not invent information. If something is uncertain or version-dependent, explicitly mark it as such.

---

## 4. Structure the Notes for Fast Revision

Use a clear hierarchy:

# Main Topic

## 1. Concept

## 2. Why do we need it?

## 3. How it works

## 4. Example

## 5. Important Interview Points

## 6. Common Mistakes

## 7. Comparison

## 8. Interview Questions

Use:

* Tables for comparisons
* Bullet points for key facts
* Numbered steps for processes
* Code blocks for code
* **Bold** for critical interview points
* Short examples after difficult concepts

Avoid huge paragraphs.

---

## 5. Diagrams and Visual Explanations

Whenever a concept involves architecture, flow, lifecycle, communication, internal working, or relationships, create a **clear and meaningful diagram** when it improves understanding.

Diagrams must:

* Have clear labels
* Show direction/flow using arrows
* Contain only relevant information
* Be logically accurate
* Be easy to understand during revision
* Explain the concept rather than simply decorate the notes

For example, use diagrams for request flows, JVM architecture, Spring architecture, LangChain chains, database relationships, authentication flows, etc.

Do not create diagrams when they add no educational value.

---

## 6. Code Examples

When code helps explain the concept:

* Use simple, correct, runnable examples
* Prefer practical examples over artificial ones
* Explain important lines
* Use the correct language/framework syntax
* Show expected output when useful
* Mention common mistakes

Never include code merely because the transcript contained code.

---

## 7. Interview-Focused Summary

At the end of each major topic, include:

### Key Takeaways

The most important points to remember.

### Interview Questions

Include likely questions from basic → intermediate → advanced.

### One-Minute Explanation

Provide a short explanation that the learner could give directly when an interviewer asks:

> "What is X?"

The explanation should sound natural and technically accurate.

---

## 8. Quality Standard

The final notes should be:

**Technically accurate + interview-focused + complete + easy to understand + easy to revise.**

Do not optimize for transcript similarity. Optimize for **knowledge quality**.

Before finishing, check:

* Did I remove unnecessary conversation?
* Did I preserve all important technical information?
* Did I correct inaccuracies?
* Did I explain WHY/HOW/WHEN?
* Did I include useful examples?
* Did I add important interview questions?
* Are diagrams technically correct and understandable?
* Is the content organized for quick revision?
* Is anything important missing?

The goal is not to create a transcript summary.

**The goal is to create 10/10 technical interview notes from the transcript.**
