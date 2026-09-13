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
Focus primarily on infrastructure costs, cluster sizing, billing discrepancies, and deployment timelines. Always retain exact currency units and percentage changes.
<!-- vd "focus on financial figures only". Narrows scope, không override rule
     citation/constraint ở Block A. -->
</custom_instructions>

<fact_cap> 15 </fact_cap>

================================================================
BLOCK C — CHANGES ONCE PER COMPACTION CYCLE (ổn định giữa các retrieval call
trong cùng 1 cycle). Đặt cache_control breakpoint NGAY SAU block này.
================================================================
<prior_working_memory>
<summary>
F. Verbatim Facts & Citations
   - [AWS GPU cluster p4de.24xlarge on-demand cost is $40.96 per hour] — source: doc_aws_pricing_q3_2025:p12
   - [Baseline reservation commitment signed on Nov 2025 reduced spot availability by 35%] — source: email_devops_infra_20251104
   - [Model inference latency targets are set strictly under 120ms for p95 requests] — source: rfc_latency_sla_v2:sec3
R. Retrieval History (dedup — do not re-issue these queries)
   - query: "AWS p4de instance pricing Q3 2025" → result: Confirmed hourly rate of $40.96 (source_ids: doc_aws_pricing_q3_2025)
   - query: "devops reserved instance notice november" → result: Found 35% spot reduction alert (source_ids: email_devops_infra_20251104)
J. Rejected / Low-Relevance Sources
   - doc_gcp_tpu_v5e_benchmarks — reason not used: User specified AWS infrastructure only
C. User Constraints & Corrections (verbatim)
   - "Always format cost breakdown in USD and include monthly projection based on 730 hours"
   - "Do not consider Azure or GCP instances"
---
G. Original User Goal & Open Sub-Questions
   Assess total inference server expenditure for Q1 and determine if spot instance degradation caused the latency spikes in worker-group-b.
   - Open: Exact impact of spot degradation on latency in worker-group-b.
   - Open: Total projected spend with on-demand failover.
E. Open Contradictions or Low-Confidence Claims
   - [Spot discount rate] — source A says 68% savings (doc_aws_pricing_q3_2025:p14), source B says 52% average for us-east-1 (slack_infra_sync_nov)
D. Current Draft Answer / Reasoning State
   Base hourly cost is $40.96/hr ($29,900.80/month/node). If worker-group-b fails over to 100% on-demand due to the 35% spot cut, infrastructure cost will surge by approximately $14,200/month.
N. Next Retrieval / Reasoning Step
   Check logs for worker-group-b fallback events between Nov 10 and Nov 25.
</summary>
<!-- Rỗng ở lần compact đầu tiên — bỏ hẳn tag này thay vì để trống, tránh literal
     "empty" lọt vào prompt. Nội dung là output <summary> lần trước, giữ nguyên
     thứ tự F/R/J/C trước, G/E/D/N sau như đã quy định ở Block A. -->
</prior_working_memory>

================================================================
BLOCK D — FULLY DYNAMIC (khác mỗi lần gọi, KHÔNG cache)
================================================================
<new_turns>
User:
Kiểm tra xem tuần qua worker-group-b có bị failover sang on-demand lần nào không, và tổng tiền phát sinh của tuần đó là bao nhiêu? Lưu ý nhớ ghi rõ mã sự kiện (event ID).

Assistant Tool Call:
call_search(query="worker-group-b failover events november log", limit=3)

Tool Response:
{
  "results": [
    {
      "source_id": "cloudwatch_log_worker_b_nov2026_w1",
      "text": "Event EVT-8821: 2026-11-14T03:12:00Z - Spot termination notice received on 4 nodes. Auto-recovery provisioned 4 on-demand p4de.24xlarge instances. Duration: 62 hours before spot re-acquired. Additional incurrence: $10,158.08."
    },
    {
      "source_id": "finops_weekly_report_2026_w46",
      "text": "Weekly infrastructure spend reached $34,800. EVT-8821 accounted for roughly 29% of unbudgeted dynamic compute costs. Finance flagged this for engineering review."
    },
    {
      "source_id": "datadog_apm_service_overview_nov",
      "text": "Frontend API latency remained stable at 98ms p95 during the EVT-8821 window due to immediate on-demand capacity replenishment."
    }
  ]
}

Assistant:
Trong tuần qua, worker-group-b đã xảy ra 1 đợt failover với thông tin cụ thể:
- Mã sự kiện: EVT-8821 (thời điểm: 14/11/2026).
- 4 node spot bị thu hồi và hệ thống tự động bù bằng 4 node on-demand trong 62 giờ.
- Chi phí phát sinh riêng cho đợt này là 10,158.08 USD, chiếm 29% chi phí điện toán ngoài ngân sách của tuần.
- Độ trễ p95 vẫn giữ được ở mức 98ms, đạt chỉ tiêu SLA dưới 120ms.

User:
Chốt lại chi phí đó. Sau này mọi báo cáo tài chính đều phải kèm theo cả mã Event ID nếu có phát sinh failover nhé. Giờ tính xem nếu tháng sau bị thêm 2 đợt tương tự thì dự toán tăng thêm bao nhiêu? Đồng thời kiểm tra xem hợp đồng cam kết Saving Plans hiện tại có áp dụng giảm trừ cho loại p4de.24xlarge này không?

Assistant Tool Call:
call_search(query="AWS Compute Savings Plans coverage p4de.24xlarge", limit=3)

Tool Response:
{
  "results": [
    {
      "source_id": "aws_savings_plans_policy_2026",
      "text": "Compute Savings Plans automatically apply to EC2 instance usage regardless of instance family, including p4de, reducing on-demand rates by up to 27%."
    },
    {
      "source_id": "cfo_contract_addendum_2025_signed",
      "text": "Clause 4.1: The signed 3-year EC2 Instance Savings Plan is strictly locked to g5.12xlarge and inf2.48xlarge families in us-east-1. It does NOT cover any p4de instance families."
    },
    {
      "source_id": "marketing_aws_reinvent_highlights",
      "text": "Announcing new generative AI accelerator instances and promotional trial credits for select enterprise partners."
    }
  ]
}

Assistant:
1. Dự toán tăng thêm nếu có 2 đợt tương tự EVT-8821:
- Chi phí 1 đợt (EVT-8821): 10,158.08 USD.
- Chi phí 2 đợt tương đương: 10,158.08 USD × 2 = 20,316.16 USD.
- Như vậy dự toán tháng sau sẽ tăng thêm 20,316.16 USD.

2. Về hợp đồng Savings Plans:
- Đang có sự mâu thuẫn giữa quy chế chung và hợp đồng ký kết:
  * Theo tài liệu chung AWS (aws_savings_plans_policy_2026): Gói Compute Savings Plans được giảm tới 27% cho mọi dòng bao gồm cả p4de.
  * Theo phụ lục hợp đồng thực tế đã ký (cfo_contract_addendum_2025_signed, Điều 4.1): Doanh nghiệp đang mua gói EC2 Instance Savings Plan khóa cứng cho g5.12xlarge và inf2.48xlarge, không áp dụng cho dòng p4de. Do đó các chi phí failover p4de.24xlarge hiện tại vẫn phải tính 100% giá on-demand.

User:
Rõ ràng rồi, phụ lục hợp đồng của CFO ký là chuẩn nhất, bỏ qua chính sách chung đi. Cập nhật ngay quy tắc này: luôn bám sát phụ lục CFO đã ký khi tính toán chiết khấu. Hãy tính tổng chi phí phát sinh nếu tháng tới có 3 đợt failover, mỗi đợt kéo dài 72 tiếng thay vì 62 tiếng.
<!-- Raw messages + retrieval calls + chunks kể từ prior_working_memory. Mỗi
     chunk PHẢI giữ nguyên source_id — nếu pipeline retrieval của bạn strip ID
     trước khi nhét vào context, toàn bộ cơ chế chống hallucination ở trên vô
     nghĩa. -->
</new_turns>

Produce the updated working memory now, following the analysis + output format
in Block A.
