---
title: "OCR to ical, or: My first attempt of doing something useful with generative AI"
date: 2024-09-20
tags:
  - llm
  - gen-ai
  - ocr
  - ics
tootId: "113211324580010966"
draft: false
---

TLDR: [ocrical](https://ocrical.romanboehm.com/). 😎

## Introduction

As a person living in Germany, you get quite a lot (read: too much) of schedules on paper or, if you're lucky, as PDFs. E.g. my daughter's school sends out these newsletters once a quarter or so with dates for the parent-teacher conferences, school plays, extracurricular activities, deadlines, you name it.

In the past, I went ahead and entered all these dates manually into my Google calendar since OCR alone couldn't make sense of the unstructured data. And there's not a program I could write that would be able to do so by itself -- there's just too many different layouts.

The one thing that _would_ be able to make sense of it is an LLM.

## ocrical

[_ocrical_](https://ocrical.romanboehm.com/), as in _OCR to ical_ will take your image or pdf, have an LLM run OCR on it and then generate an iCal file from the schedule-related data. It is heavily inspired by / stole shamelessly from both Simon Willison's [ocr](https://github.com/simonw/tools/blob/main/ocr.html) and [gemini-chat](https://github.com/simonw/tools/blob/main/gemini-chat.html).

Why Gemini? I had the example above and I wanted to do the project without a backend, only in the browser. Gemini has the JS library to do just that. Plus, it seems to be good enough at both OCR and generating the ical file. Last but not least: I don't have a clue which of the LLMs does both these tasks best.

## Conclusion

I fed the first few letters into ocrical: It did _well enough_. Especially considering how crude the data often is.

I'm not too happy about feeding it two prompts, and hope to converge on a single prompt in future, but I wanted to get this out now and be able to use it myself. Maybe you'll find it useful, too.
