---
title: "Diablo II binary parser using Go"
date: "2022-02-27"
tags: ["diablo-ii"]
draft: false
---

In 2016 I joined a private Diablo II server called Slashdiablo. It's run by a few Diablo enthusiasts and
alot of code and tools had been written to keep the place running. As I started playing and engaging
with the community on Discord I quickly noticed user interfacing services I was missing from modern 
games such as an easy way to see player rankings or the ability to check character builds online.

I started to dig into the architecture of Diablo II and how they were saving the character data I was 
interested in. It turns out there were some people back in the day (early 2000's) who spent a lot of time
reverse engineering the game, but since then a lot of that work had been lost. To be fair it's been almost
20 years since the work was done but I did find some clues on how to proceed.

It turns out Diablo II stores all character data in tightly packed binary files called `.d2s` files.
This binary layout was designed when every bit mattered so they didn't waste a single bit packing data.
