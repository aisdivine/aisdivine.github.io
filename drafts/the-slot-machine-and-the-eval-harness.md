# Video Gen: The Slot Machine and the Eval Harness

*Category: interactivity · September 11, 2026*

**Dek:** Video generation is a slot machine: every pull is cheap, random, and occasionally a jackpot. I spent a couple of weekends running a one-person TV studio out of my terminal, and almost none of the engineering went into generating video. It went into the harness that decides whether a pull was any good — because you can't beat a slot machine, but you can get very good at knowing when it paid out.

---

Everyone starts at the slot machine. You type a prompt, a model hands you eight seconds of video, and for about a day that feels like the product. It isn't. The generator is a button anyone can press. The thing I actually built — the thing that took all the time and taught all the lessons — was the instrument panel around the button.

I've argued that [when software is cheap, evaluation is the product](#evaluation-is-the-product). That post was the theory. This one is the weekend-sized proof. My background is in human-centered computing — think HCI, but more so centered on the human.

One honesty clause before the numbers, because the numbers are about to sound authoritative: everything here is field notes, not a paper. The sample sizes are single digits to low dozens, and the ground truth throughout is one person's eye — mine. For a one-person studio that's not a flaw, it's the spec: my taste is the product definition. But you should read every "6 for 6" below as *a probe*, not *a result*.

## The Weekend Project

The project: micro-dramas. Sixty-second vertical episodes, generated end to end — stills, video, voices, sound, the cut — from a show bible in a git repo, driven by scripts. One show is about a drowned woman living inside an octopus while her fiancé keeps calling her voicemail. Another is a toddler show whose cast is my kid's actual toys: an orange inflatable bounce dino and a green wooden pull-along caterpillar, photographed on the living-room floor and turned into recurring characters.

Finished episodes cost between $1.30 and $6 each. When an episode of television costs less than a sandwich, generation is not the scarce thing. Knowing whether it's *good* is.

## The Machine Is a Slot Machine, Literally

Things I believed about generative video that a few sub-dollar probes disproved:

- **Seeds don't reproduce clips.** Same prompt, same seed, different video. There is no replay button. Every pull is a fresh pull.
- **The content filter is a slot machine too.** The same prompt passes, then fails, then passes.
- **Models drift under motion.** One model turned the green caterpillar magenta mid-shot, once, for no reason it ever repeated.

So the right mental model is not "compiler," it's casino: you can't re-run a result, you can only characterize the distribution and pull again. Which flips the interesting engineering question. It's not *how do I generate a good clip?* — you don't control that. It's *how cheaply, quickly, and reliably can I recognize one?*

That's an eval harness. Here's what building one taught me.

## Learning 1: A Judge Lies Fluently Until You Score It

The obvious move is model-as-judge: have a vision model inspect every generated still and flag defects before you spend money animating them. I did that. It narrated its reasoning beautifully. It sounded rigorous. And when I finally scored it against a deck of cases I'd labeled by eye, it was **52% accurate with a 39% false-alarm rate** — worse than a coin flip on the cases that mattered, and every false alarm spent real money re-rolling work that was already correct.

The instinct is to reach for a smarter model. I tried: the smarter model scored **48%**. What actually fixed it was three iterations on the rubric — the questions, not the brain — which took the *same* judge to 74% accurate with 9% false alarms. The rubric was the lever. It usually is.

Two rules fell out. First: no judge gates spending until it's been scored blind against labeled cases, in both directions — a judge that narrates well and concludes badly reads as trustworthy right up until you score it. Second, and nobody warns you about this one: **the deck rots.** When I re-audited my labeled cases weeks later, a handful of labels had quietly gone stale — approvals from before the show's canon settled, flags for defects that were no longer defects. The eval needs its own eval, on a schedule.

## Learning 2: Narrow the Question Until the Answer Is a Fact

The single most reusable trick I found. I asked a judge "did this beat perform?" and it went 4 for 6 — wrong in both directions. I rewrote the question as "in the last frame, is the man LYING DOWN or RISEN?" and it went 6 for 6.

Same model, same clips. The difference: the second question has a factual answer from a fixed set, and the *judgment* — was that the beat we wanted? — moved into code that compares the answer to what the script required. One categorical question, no other job, decision made by arithmetic.

This rescued every unreliable judge I had. Identity checking failed when bundled with framing and hands; asked alone — this face, this reference portrait, categorical evidence only — it went 4 for 4, and caught a shot where the model had quietly recast my lead. Small n's, yes. But the direction was the same every time I made the move, and never the same when I didn't.

## Learning 3: The Composite Score Is Not a Gradient

I gave a judge a rubric and let it emit an overall score out of 10, then tried to hill-climb the score. Three consecutive builds — each one fixing defects the judge itself had named — scored **6.4, then 6.1, then 5.9**. Meanwhile the same judge scored a byte-identical file ±0.5 apart on consecutive runs.

The noise is bigger than the effect of any single fix. Descending that gradient is descending static. But the judge's *defect list* — "the gavel appears from nowhere in shot 7" — was specific, reproducible, and almost always right. So: take a judge's defects, never its mean. The number is decoration. The list is the signal.

## Learning 4: Gates Measure Broken, Not Good

The humbling one. I built deterministic gates — luma bands, cut-plan assertions, audio levels — plus two independent judge passes, and produced a cut that passed everything. A human watched it for ten seconds and said: *"this looks like all the other AI-generated crap."* He also spotted a wrong face, a looped clip, and a hallucinated prop. No instrument had flagged any of it.

Automated checks answer "is it broken?" They cannot answer "is it good?" — and while I optimized the legible metrics, taste stayed unmeasured, so the result scored fine and was unwatchable. Whether taste is unmeasur*able* or just unmeasured-so-far is a question I'm still paying to find out; every learning above is a piece of taste that stopped being vibes and became a check. But the honest current answer is: a clean gate is a precondition, not a verdict. Every green dashboard gets paired with a human actually watching the thing, and the harness's real job is to make that human's ten seconds count — surface the three real decisions, not the forty checks that passed.

A scoping note the first draft of this post got wrong: which error costs more depends on what the judge gates. When it gates *re-rolls*, false alarms are the expensive error — they burn money rejecting correct work. When it gates *shipping*, a miss is the expensive error — a miss is how the wrong face reaches an audience. Same judge, opposite economics. Decide which gate you're building before you decide what to tolerate.

## Learning 5: Probe the Expensive Assumption for a Dollar

The octopus show needed its lead — an octopus — to act. I ran a ladder of ~$0.25 probes to find out what the model could actually perform: end frames don't fix locomotion (and can make it worse), forbidding the canonical motion went 0 for 2, rewording the state change went 0 for 3. What worked: **the skin**. Darkens, pales, roughens; the pupil narrows. That is the entire usable acting range of a generated octopus, and the show got rewritten to be performable within it — which made it better.

Total cost of learning the true constraint: a few dollars. Meanwhile I once spent a full day architecting around the belief that human faces were too unreliable to generate — and a single $2 probe disproved it. The general form: any assumption expensive enough to design around is cheap enough to test first.

## Learning 6: Eval the Operator Too

The quiet compounding trick: before any rebuild or reshoot, write down what you expect to happen, then score the prediction against the measurement. My first ledger entry went 6 for 7. Some later ones went worse — and those were the interesting ones, because a wrong prediction locates a wrong belief *before* it spends money. Once the harness exists, pointing it at your own judgment is nearly free.

By the end this went one step further: scoring a show *concept* for generatability — how much of it is deterministic (comps, text on screens, audio) versus rolled fresh on the slot machine — before generating a single frame. The show that scored high shipped 10 of 12 stills on the first pull; the pilot cost $1.30. The show that scored low took twenty-plus versions of some shots. The cheapest eval is the one you run before the pixels exist.

## Whose Taste Is the Ground Truth?

Every number above bottoms out in one person's eye, and for the micro-dramas that's the honest spec — a one-person studio ships one person's taste. But the toddler show exposes the limit of that answer, because it has a real audience: one viewer, about two and a half feet tall, who cannot be rubric'd, has never respected a composite score, and renders a verdict by either watching or walking away. Every judge and gate in my harness is a proxy for a decision that viewer makes in three seconds, and the day my instruments and the audience disagree, the instruments are wrong. That's not a footnote — that's the direction all of this has to grow. The harness I built converges on *my* eye; the next one has to converge on the room's.

## Facts That Cost Money to Learn

A short ledger, in memoriam:

- There is no red octopus in cold deep water. The color grade had to carry what the biology wouldn't.
- Every clip prompt now carries a COLOUR LOCK clause, because of the magenta caterpillar.
- Character voices must come from a TTS with fixed voice IDs. Let the video model speak and your character gets a new voice every clip.
- One provider bills a five-second minimum. Ask me how I know.

I keep these in the repo the way a kitchen keeps a burn chart. Each entry is small; the discipline — every mistake gets named, turned into a mechanism, and tested so it can't recur — is the actual asset.

## The Harness Is the Studio

The generator is a commodity — a slot machine anyone can rent by the pull. What turned mine into a studio was everything wrapped around it: judges scored before they're trusted, asking narrow factual questions; gates that know they only measure broken; probes that price out assumptions before designs do; a ledger that evals the operator along with the model; and a standing appointment to re-label the deck.

My hypothesis — held at the confidence two weekends buys — is that none of this is video-specific. So the next time a generative system hands you something impressive, the question worth engineering is not "can it make this?" You already know it can, sometimes. The question is: *how would you know, cheaply and repeatably, whether this pull was the jackpot?* Build that first. It's more fun than it sounds, and it's the only part the slot machine can't do.

---

## Further Reading

- Zheng, L., et al. (2023). ["Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena."](https://arxiv.org/abs/2306.05685) NeurIPS. — The canonical look at model-as-judge agreement and its biases; the reason to score your judge before trusting it.
- Shankar, S., Zamfirescu-Pereira, J.D., Hartmann, B., Parameswaran, A., & Arawjo, I. (2024). ["Who Validates the Validators? Aligning LLM-Assisted Evaluation of LLM Outputs with Human Preferences."](https://arxiv.org/abs/2404.12272) UIST. — Criteria drift, formalized: your rubric changes as you read outputs, so the eval needs its own eval. Learning 1's second rule, with a methodology.
- Clark, E., August, T., Serrano, S., Haduong, N., Gururangan, S., & Smith, N.A. (2021). ["All That's 'Human' Is Not Gold: Evaluating Human Evaluation of Generated Text."](https://aclanthology.org/2021.acl-long.565/) ACL. — The human eye is an uncalibrated instrument too; why "labeled by eye" deserves its own scrutiny.
- Goodhart's law, via Strathern, M. (1997). ["'Improving ratings': audit in the British University system."](https://doi.org/10.1002/(SICI)1234-981X(199707)5:3%3C305::AID-EURO184%3E3.0.CO;2-4) — When a measure becomes a target, it ceases to be a good measure. Learning 4, formalized in 1997.
