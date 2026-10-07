# Prompts and rules (verbatim from app/index.html, rules v1.1)

Placeholders like `${cardText(card)}` are filled in per case. The router is plain JavaScript, not an AI.

## Constants and consulting method

```js
const CATS = ["demand","costs_inputs","pricing_competition","regulation_legal","governance_leadership","capital_financing_valuation","operations_supply_chain","strategy_portfolio","macro_geopolitics","technology_shift","accounting_one_off","market_expectations"];
const MACRO = "Macro backdrop, mid-2026: a US–Iran conflict since around March 2026 has kept crude oil high (near or above $100 at times) and disrupted Gulf shipping and freight; US Treasury yields reached multi-year highs and rate-hike expectations returned; broad US tariffs remain in force on many trading partners; AI infrastructure spending is booming and memory-chip prices rose sharply in H1 2026; investors worry AI will disrupt software and IT-services businesses.";
const METHOD = `Diagnose like a senior strategy consultant. Company statements and press coverage are partly PR, so read between the lines. Connect the event to every stakeholder: the company's own targets and vision, its shareholders, its competitors and the peers it imitates, its suppliers and customers, regulators and government, and the wider world (macro, geopolitics, technology shifts). Look for the pattern that links them. Prefer specific drivers ("raw material costs rose sharply") over generic phrases ("macro headwinds", "execution issues").`;
```

## Prompts

```js
const PROMPTS = {
  solo: (card) => `${METHOD}

${cardText(card)}

Task: using only information that would have been available up to the event date, identify the three most likely causes of this event, ranked. If the figures or the macro backdrop need interpreting, you may call ask_context_analyst (at most once). If the card gives too little to diagnose responsibly, set insufficient_context to true.

Reply with only JSON in this shape:
{"insufficient_context": false, "causes": [{"rank": 1, "cause": "specific driver, max 20 words", "category": "one of ${CATS.join(", ")}", "why": "max 25 words tied to this company's industry, size, geography or finances"}, {"rank": 2, ...}, {"rank": 3, ...}], "recommendation": "max 40 words, specific to this company"}`,
  generate: (card) => `${METHOD}

${cardText(card)}

Task: an expert will review your candidates. List 10 distinct candidate causes of this event, in random order, without ranking them or signalling which you favour. Each needs a reason tied to this company's industry, size, geography or finances. Use only information available up to the event date. You may call ask_context_analyst (at most once) if figures or the macro backdrop need interpreting.

Reply with only JSON: {"candidates": [{"cause": "max 18 words", "category": "one of ${CATS.join(", ")}", "why": "max 20 words"}, ... 10 items]}`,
  rank: (card, kept, killed) => `${METHOD}

${cardText(card)}

An expert reviewed candidate causes. They KEPT these (some may be their own additions):
${kept.map((k,i)=>`${i+1}. ${k.cause}${k.added?" (expert's own)":""}`).join("\n")}
They KILLED these (do not use them):
${killed.length?killed.map(k=>"- "+k.cause).join("\n"):"- none"}

Task: rank the kept causes and return the top three (fewer if fewer were kept), chosen only from the kept list, then give one recommendation specific to this company.
Rule: causes marked (expert's own) were added by a domain expert after reviewing the case. Every one of them MUST be in your top three (if there are more than three, the top three are all expert causes). You decide their order on the evidence. Keep the expert's meaning; you may tidy the wording. Mark them "from_expert": true and use "why" to explain how each fits this company.

Reply with only JSON: {"causes": [{"rank": 1, "cause": "...", "from_expert": false, "category": "one of ${CATS.join(", ")}", "why": "max 25 words"}], "recommendation": "max 40 words"}`,
  analyst: (card, q) => `You are a context analyst supporting a business diagnostician. Work only from the case card below and information available before the event date.

${cardText(card)}

Question from the diagnostician: ${q}

Use the calculator tool for any arithmetic. Reply in at most 80 words: the key numbers and what they imply. No preamble.`,
  judge: (key, answers, cands) => `You are a strict scorer. Match each candidate answer to the answer key.

ANSWER KEY (causes stated by the company or reputable reporting):
${key.map((k,i)=>`K${i+1}. ${k}`).join("\n")}

RULES
1. Same driver in any wording is a match ("input cost inflation" matches "raw material costs up 63%").
2. A generic phrase that fits any company is not a match ("macro headwinds", "market conditions", "execution issues").
3. The right symptom with the wrong driver is not a match ("costs rose because of a strike" when the key says commodity prices).
4. Judge each line on its own.

ANSWERS TO JUDGE (labels are random and carry no meaning):
${answers.map(a=>a.lines.map((l,i)=>`${a.label}.${i+1} ${l}`).join("\n")).join("\n")}
${cands.length?`\nCANDIDATE LIST L:\n${cands.map((c,i)=>`L.${i+1} ${c}`).join("\n")}`:""}

Reply with only JSON: {"matches": [{"id": "A.1", "key": "K1 or null", "reason": "max 12 words"}, ...]} with one entry for every line above, including every L line.`
};
```

## Case card format

```js
function cardText(c){
  return `CASE CARD
Company: ${c.company}
Event date: ${c.date}
What happened: ${c.event}
Size: ${c.size}
Industry: ${c.industry}
Geography: ${c.geography}
Ownership: ${c.ownership}
Financial position: ${c.financial_position}
Technology tailwind or headwind: ${c.tech}
Recent company context (before the event): ${c.recent}
${MACRO}`;
}

```

## Router

```js
function route(card, attempts){
  const f=card.flags||{}; const flagNames=Object.entries({private:"private company",regulated:"regulated sector",unusual_financial:"unusual finances",leadership_change:"recent leadership change"}).filter(([k])=>f[k]).map(([,v])=>v);
  const abst=attempts.filter(a=>a?.out?.insufficient_context).length;
  if(abst>=2) return {lane:"human", why:`${abst} of 3 AI attempts said the context is insufficient`, agree:0, flags:flagNames};
  const tops=attempts.map(a=>a?.out?.causes?.[0]?.category).filter(Boolean);
  const counts={}; tops.forEach(c=>counts[c]=(counts[c]||0)+1);
  const agree=Math.max(0,...Object.values(counts));
  const need = flagNames.length===0?2 : flagNames.length===1?3 : Infinity;
  if(agree>=need) return {lane:"ai", why:`${agree} of 3 attempts agree on the top cause type`+(flagNames.length?` (bar raised to 3/3 by: ${flagNames.join(", ")})`:""), agree, flags:flagNames};
  return {lane:"pair", why: flagNames.length>=2?`two or more flags: ${flagNames.join(", ")}`:`only ${agree} of 3 attempts agree on the top cause type`+(flagNames.length?`; flag raised the bar: ${flagNames.join(", ")}`:""), agree, flags:flagNames};
}

```
