# Week 4 — Polish, Documentation + Interview Prep

> Platform: PC  
> Sessions: 5 weekdays × 8 hrs + optional weekend reading (2 hrs/day)

---

## Goals

- Make your GitHub repo presentable to an interviewer
- Sharpen your technical narrative and talking points
- Practice answering embedded interview questions cold
- Submit the ADI application

---

## Sessions

### Session 4.1 — GitHub Cleanup + Documentation
**Focus:** Your GitHub is your portfolio — make it count

Tasks:
- Every project has a `README.md` with: what it does, hardware used, how to build/flash, what you learned
- Remove dead code, commented-out blocks, debug hacks
- Consistent commit history — no "fix", "asdf", "wip" messages
- Add a top-level repo README that links to each week's work with a one-line description
- Add wiring diagrams or photos where relevant (even phone photos are fine)

**Checklist:**
- [ ] Week 0: C/C++ exercises — all compile cleanly
- [ ] Week 1: ESP-IDF peripheral projects — all flash and run
- [ ] Week 2: FreeRTOS + Linux drivers — all documented
- [ ] Week 3: Zephyr projects + logic analyzer captures included
- [ ] Top-level README links everything

---

### Session 4.2 — Talking Points + Technical Review
**Focus:** Knowing what you built well enough to defend it in an interview

For each project, be able to answer:
- What does it do?
- Why did you make the design decisions you made?
- What broke and how did you debug it?
- What would you do differently?

**Prior experience talking points (from your firmware background):**
- Network protocol implementation (LPD, FTP, SMB, SSL/TLS, IPP, HTTP/2) in C on Linux-based embedded hardware
- Full ownership: design docs, implementation, testing, bug resolution
- Technical lead on new printer project at i-BRIDGE Japan
- CI/CD maintenance (GitLab, Jenkins)

**Gap responses — practice these out loud:**
- RTOS/Zephyr: "I hadn't used Zephyr commercially, so I spent the last month building [X] with it on ESP32..."
- SPI/I2C/UART: "My protocol experience was at the network layer. I've since done hands-on work with SPI and I2C — [describe your projects]..."
- Lab equipment: "I set up a logic analyzer for this training and used it to capture and decode SPI/I2C traffic — [describe what you found]..."

---

### Session 4.3 — Mock Technical Questions: Systems + C/C++
**Focus:** Cold answers to common embedded interview questions

Practice answering these without notes:

**C/C++**
- What does `volatile` do and when do you use it in embedded?
- What's the difference between `const` and `volatile const`?
- Explain RAII and why it matters in embedded C++
- What's priority inversion and how do you prevent it?
- What does `static` mean in different contexts in C?
- How do you prevent stack overflow in an embedded system?
- What's the difference between a mutex and a semaphore?

**Systems**
- Walk me through what happens from reset to `main()` on a microcontroller
- What's a memory-mapped peripheral and how do you access it in C?
- What's the difference between bare-metal and RTOS programming?
- When would you choose polling over interrupts?
- What's DMA and when would you use it?

---

### Session 4.4 — Mock Technical Questions: Protocols + RTOS
**Focus:** Hardware protocol and RTOS questions

Practice answering these without notes:

**Protocols**
- Explain SPI — what are CPOL and CPHA?
- What's the difference between I2C and SPI? When do you choose one over the other?
- How does I2C addressing work? What happens on a NACK?
- What is UART and how do you configure baud rate?
- What is CAN bus and why is it used in automotive/industrial systems?

**RTOS / Zephyr**
- What is a task/thread in an RTOS?
- How does the FreeRTOS scheduler decide which task to run?
- What is Zephyr's device model and how does a driver bind to hardware?
- What is a device tree and why does Zephyr use it?
- How do you share a peripheral (e.g. SPI bus) safely between multiple tasks?

**Linux Drivers**
- What's the difference between kernel space and user space?
- What is a character device driver?
- How does a Linux kernel driver bind to hardware described in the device tree?

---

### Session 4.5 — Buffer + Apply
**Focus:** Catch up, final review, submit

Tasks:
- [ ] Finish anything incomplete from Sessions 4.1–4.4
- [ ] Final pass on GitHub — read your own READMEs as if you're the interviewer
- [ ] Tailor CV summary to match ADI job description language
- [ ] Write cover letter (optional but recommended — emphasize firmware background + active upskilling)
- [ ] Submit application

---

## Weekend Reading

| Day | Topic |
|-----|-------|
| Sat | Browse ADI's GitHub (analog-devices-open-source); look for Zephyr or Linux driver contributions you could comment on in an interview |
| Sun | Light review; rest before interviews |

---

## Interview Mindset

- Your firmware background is real embedded experience — own it
- You've spent a month doing hands-on work with hardware — you have projects to point to
- When you don't know something: "I haven't used that commercially, but here's how I'd approach it..."
- ADI cares about fundamentals, debugging ability, and communication — not just résumé checkboxes
