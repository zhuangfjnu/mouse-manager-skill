---
name: manage-mouse-colony
description: Use when the user asks to update or query mouse colony registration tables, add pups, record breeding/litter events, calculate maturity/weaning dates, preserve breeding history, or says "数据看板" to open the local mouse colony dashboard.
---

# Manage Mouse Colony

Use this skill for mouse colony spreadsheet maintenance. Treat the individual registration sheet as the source of truth once breeding information has been merged into it.

## First Use In A Conversation

On the first request in a conversation that triggers this skill, obtain the online registration-document address before continuing. If the user has not already supplied one, ask: `请提供小鼠个体登记在线文档地址（如 KDocs 链接）。`

- Record the supplied address as the source reference for the current conversation.
- Ask only once per conversation unless the user replaces the source document.
- The address does not authorize edits to the online document. Continue to use the local XLSX/CSV workflow unless the user explicitly requests an online-table edit.

## Data Dashboard Command

When the user's entire request is `数据看板`, locate the newest `.xlsx` file supplied in the current conversation. Run `scripts/import-conversation-xlsx.mjs` from `/Users/mac/Documents/Codex/2026-07-11/wps-https-www-kdocs-cn-l-3/mouse_colony_dashboard` with the bundled Node runtime before opening `http://127.0.0.1:3176/` in the in-app browser. If the service is unavailable, start it from that directory with the bundled Node runtime. On successful import, refresh the dashboard. If no current-conversation attachment exists or validation fails, do not alter the CSV snapshot; open the existing dashboard and state that a valid current-conversation XLSX is required. Do not modify the XLSX attachment or KDocs source table for this command.

## Today Tasks Command

When the user's entire request is `今日待办` or `今天有什么任务`, run `scripts/today-tasks.mjs` from `/Users/mac/Documents/Codex/2026-07-11/wps-https-www-kdocs-cn-l-3/mouse_colony_dashboard` with the bundled Node runtime. Use its JSON output to reply in this exact section order: `可配对资源`、`可剪尾`、`单独一笼`、`需分笼`、`信息不全`. Include the report date and show `无` for an empty section. Do not open the dashboard, modify local files, modify the XLSX attachment, or modify KDocs for this command.

## Core Rules

- Preserve history. Add new columns or new rows for new events; do not overwrite previous pairing, litter, parentage, or date records unless the user explicitly asks to correct an error.
- Prefer one main worksheet named `小鼠个体登记表`. If a separate breeding worksheet exists and the user wants consolidation, copy its history into the individual sheet before deleting the breeding worksheet.
- Use exact dates in `YYYY/M/D` format unless the sheet already uses another clear format.
- Interpret relative dates using the active system date/timezone, and state the resolved absolute date when it matters.
- For web spreadsheets, verify visible columns after paste operations. WPS/KDocs can keep the active sheet or horizontal position in surprising places.

## Individual Sheet Schema

Maintain these leading columns in order when possible:

1. `目标号`
2. `基因型`
3. `出生日期`
4. `性成熟日期`
5. `性别`
6. `笼位坐标`
7. `亲代1`
8. `亲代2`

After those, keep repeated history columns:

- `产仔日期`, `产仔日期2`, `产仔日期3`, ...
- Breeding-history groups, when needed: `配对日期1`, `配对状态1`, `繁殖笼位1`, `产仔数量1`, `预计分笼日期1`, then `配对日期2`, ...

Do not keep a `状态` column if the user has asked to replace it with parentage fields.

## Date Calculations

- `性成熟日期` = `出生日期 + 56 days`.
- `预计分笼日期` = `产仔日期 + 20 days`.
- When a litter is weaned and pups are added, use the litter's `产仔日期` as the pups' `出生日期`.
- Breeding projection defaults when exact colony data are absent: gestation/litter birth = pairing date + 19–21 days; weaning/genotyping checkpoint = litter date + 20 days; next-generation pairing can begin at birth date + 56 days.

## Breeding Projection for Target Genotypes

When the user asks how long it will take to obtain a target genotype from a proposed pair:

- Use the current date as the pairing date unless the user gives another date, and state the exact date used.
- Read each parent genotype from the sheet. Do not assume an unlisted allele is present.
- If only one parent carries a floxed allele and the other parent is not recorded as carrying it, direct offspring cannot be flox homozygous; they can be at most heterozygous for that allele.
- To estimate a two-generation plan for a target such as `wnt1-cre + Bmp4flox` homozygous:
  1. Parent cross: carrier/heterozygote × Cre carrier gives F1; select F1 that carry both Cre and the floxed allele.
  2. F1 birth = parent pairing + 19–21 days.
  3. F1 weaning/genotyping = F1 birth + 20 days.
  4. F1 can be re-paired at birth + 56 days.
  5. F2 birth = F1 pairing + 19–21 days.
  6. F2 confirmation = F2 birth + 20 days.
- Report both milestones: earliest target birth date and earliest practical confirmation date after weaning/genotyping.
- For heterozygote × heterozygote at a single floxed locus, estimate homozygous offspring probability as 25%. Cre inheritance depends on recorded Cre zygosity; if unknown and treated as common Cre heterozygous transmission, state that the target probability is an estimate and may be lower than or around the flox-homozygous fraction depending on the selected F1 breeders.

## Breeding Events

When the user records a pairing and litter for a pair:

- Add a new breeding-history group for both parent rows.
- Add/update `产仔日期N` on parent rows, but do not record pairing dates in the simple litter-date columns.
- If a litter count is provided, place it in the matching `产仔数量N`. If it is not provided, leave it blank rather than inventing a value.
- Keep cage positions under `笼位坐标` for current individual housing, and under `繁殖笼位N` for historical breeding cages.

## Adding Pups

Append pup rows to `小鼠个体登记表`; do not overwrite existing individuals.

For each pup row, fill:

- `目标号`: ear tag or ID, such as `Z1`.
- `基因型`: use exactly the user-provided genotype text, such as `DTA（HE）` or `DTA（HO）`.
- `出生日期`: litter date.
- `性成熟日期`: birth date + 56 days.
- `性别`: user-provided sex.
- `笼位坐标`: new cage position after weaning.
- `亲代1` and `亲代2`: parent IDs, such as `M9` and `M10`.

If the user says a range like "Z1 to Z5" but lists only specific IDs, follow the explicitly listed IDs and avoid creating missing IDs unless clearly requested.

## Queries

Answer counts from the current sheet state.

- Genotype counts: count rows whose `基因型` contains the requested genotype string, allowing full-width parentheses.
- Available breeding females: count rows matching the genotype, `性别 = 雌性`, and `性成熟日期 <= current date`. Exclude rows with future maturity dates.
- Birth/litter count for an individual: count non-empty `产仔日期*` cells in that individual's row.

When the query depends on the current date, include the exact date used in the answer.

## Verification

After edits, visually verify:

- Headers are aligned with values.
- Parentage columns did not shift into date columns.
- Litter-date columns contain only litter dates, not pairing dates.
- Historical breeding groups remain intact after deleting any old breeding worksheet.
