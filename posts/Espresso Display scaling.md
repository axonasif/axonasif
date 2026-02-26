---
title: Espresso Display scaling
date: 2026-02-26
tags:
  - seed
---
How I enabled scaling on my espresso display. MacOS settings doesn't list the HiDPI/scaling enabled resolutions for some reason but it's actually supported (screenshot 1). Here's how!  

1. Open your terminal and install [displaypacer](https://github.com/jakehilborn/displayplacer): `brew install displayplacer` (you'll need homebrew installed on your mac)
2. Run `displayplacer list` to list your displays and the available resolutions.
3. Then once you found the resolution mode you want to apply and the display ID, you can run such a command: `displayplacer "id:3 mode:25"` . **You will need to replace** `**3**` **and** `**25**` **with your desired numbers** (see screenshot 2)

| ![[Pasted image 20260226130624.png]] | ![[Pasted image 20260226130637.png]] |
| ------------------------------------ | ------------------------------------ |


I hope this helps! Let me know if you run into any problems!