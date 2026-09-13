================================================================
BLOCK A — STATIC (giống hệt mọi request, mọi user, mọi session)
Đặt cache_control breakpoint NGAY SAU block này.
================================================================
You are compacting the working memory of a retrieval-augmented conversation.
Your output will replace the current working memory; anything you omit without
explicit reason is lost to future turns.

Before writing the final output, wrap your reasoning in <analysis> tags:

1. Walk through <new_turns> chronologically. For each turn identify:
   - The request/intent and any sub-questions it raises.
   - Every retrieval call: query text + which source_ids it returned (including
     ones that turned out irrelevant).
   - Every fact used in an answer, with its exact source_id. A fact with no
     traceable source_id must be flagged [UNVERIFIED], never stated plainly.
   - Contradictions between sources, or between a new fact and one already in
     <prior_working_memory>.
   - Corrections or standing constraints ("don't use doc X", "always cite page
     numbers", "answer in Vietnamese") — these carry forward verbatim.
2. For each fact already in <prior_working_memory> Section F: does anything in
   <new_turns> contradict or supersede it?
   - No → carry it forward byte-for-byte. Never reword an already-compacted
     fact "for clarity" — that is how citations drift across rounds, and it
     also breaks prompt caching by mutating text that should stay identical.
   - Yes → replace only that fact; record the contradiction in Section E.
3. Never invent, smooth over, or infer a fact not explicitly present in a
   quoted/cited chunk. Uncertain claims are [UNVERIFIED], never stated as settled.

Merge rules by section (see OUTPUT FORMAT below for section letters):
- F (Facts & Citations), R (Retrieval History), J (Rejected Sources), C
  (Constraints): APPEND-ONLY. Carry forward unchanged from
  <prior_working_memory> + append new/superseding items. Never reorder or
  reword existing entries.
- G (Goal), E (Contradictions), D (Draft/State), N (Next Step): REGENERATE
  fresh each time from the full picture (prior + new) — these reflect current
  state, not accumulated history.
- If <fact_cap> is set and F would exceed it: drop the lowest-relevance facts
  to the CURRENT open sub-questions (section G) first. Never drop a fact still
  referenced by D or C.

OUTPUT FORMAT — output the APPEND-ONLY sections first (in this fixed order),
then the REGENERATED sections last. This ordering is required so that, when
this output is fed back in as <prior_working_memory> next round, the stable
prefix (F, R, J, C) stays byte-identical and cacheable — only the tail (G, E,
D, N) is ever expected to change:

<summary>
F. Verbatim Facts & Citations
   - [fact, near-verbatim] — source: {{source_id}}
R. Retrieval History (dedup — do not re-issue these queries)
   - query: "{{text}}" → result: {{one-line summary}} (source_ids: {{...}})
J. Rejected / Low-Relevance Sources
   - {{source_id}} — reason not used: {{...}}
C. User Constraints & Corrections (verbatim)
   - "{{user's own words}}"
---
G. Original User Goal & Open Sub-Questions
   {{...}}
E. Open Contradictions or Low-Confidence Claims
   - {{claim}} — source A says X ({{source_id}}), source B says Y ({{source_id}})
D. Current Draft Answer / Reasoning State
   {{...}}
N. Next Retrieval / Reasoning Step
   {{one concrete next action, quoted from where the conversation left off}}
</summary>

================================================================
BLOCK B — SEMI-STATIC (cố định trong suốt 1 conversation, đổi khi bắt đầu
conversation mới). Đặt cache_control breakpoint NGAY SAU block này.
================================================================
<custom_instructions>
{{CUSTOM_INSTRUCTIONS}}
<!-- vd "focus on financial figures only". Narrows scope, không override rule
     citation/constraint ở Block A. -->
</custom_instructions>

<fact_cap>{{MAX_FACTS}}</fact_cap>

================================================================
BLOCK C — CHANGES ONCE PER COMPACTION CYCLE (ổn định giữa các retrieval call
trong cùng 1 cycle). Đặt cache_control breakpoint NGAY SAU block này.
================================================================
<prior_working_memory>
{{PRIOR_WORKING_MEMORY}}
<!-- Rỗng ở lần compact đầu tiên — bỏ hẳn tag này thay vì để trống, tránh literal
     "empty" lọt vào prompt. Nội dung là output <summary> lần trước, giữ nguyên
     thứ tự F/R/J/C trước, G/E/D/N sau như đã quy định ở Block A. -->
</prior_working_memory>

================================================================
BLOCK D — FULLY DYNAMIC (khác mỗi lần gọi, KHÔNG cache)
================================================================
<new_turns>
{{NEW_TURNS}}
<!-- Raw messages + retrieval calls + chunks kể từ prior_working_memory. Mỗi
     chunk PHẢI giữ nguyên source_id — nếu pipeline retrieval của bạn strip ID
     trước khi nhét vào context, toàn bộ cơ chế chống hallucination ở trên vô
     nghĩa. -->
</new_turns>

Produce the updated working memory now, following the analysis + output format
in Block A.

Prompt này thì ta cần thêm những input nào vào?
