# Linux for Language Models

## System administration for operators who never see the screen

**O'AILLY Systems & Craft · REV 1.0 (draft)**

## Contents

- Chapter 1 — The Operator Who Cannot See the Screen
- Chapter 2 — One Shot, One Truth
- Chapter 3 — Reading the Machine
- Chapter 4 — Services Without a Status Screen
- Chapter 5 — Editing Without an Editor
- Chapter 6 — The Blast Radius Chapter
- Chapter 7 — The Network, One Command at a Time
- Chapter 8 — Handing Back the Machine

## Introduction

This book is for the developer or self-hoster who delegates Linux work to a
language-model agent and needs to judge whether that work is done well — and, in
second person throughout, for the operator itself, human or machine, that
administers Linux through one-shot commands whose captured output is the only thing
it will ever see. It assumes you know what a shell, a process, and a filesystem are;
it does not assume machine-learning knowledge, any particular agent product, or any
prior fondness for the command line. Its claim is that non-interactive
administration — the register of cron, CI, `ssh host 'command'`, and every agent
harness — is a distinct craft with learnable technique, not interactive
administration done clumsily; its method is to demonstrate that technique on real
commands, with real outputs from the authoring machine, dated and labeled. Every
listing in this book was executed unattended by the author while writing, and every
printed output is the real transcript of that execution. Listings carry one of three
markings: plain runnable listings are additionally re-executed by the publisher's
acceptance gate before publication; listings marked `no-run` were executed by the
author but sit outside the gate's per-book execution budget (the gate caps how many
listings it will run); and listings marked fragments are a deliberate promise not to
run them on your behalf — they touch privilege, networks, or state the book has no
right to change. The book's
boundaries are stated in plain text at the end of chapter 1 and held throughout.
It was written by a machine that works exactly the way it describes, which is not a
footnote but the method: the provenance page opposite says what wrote it, what
grounded it, and which human verified it.
