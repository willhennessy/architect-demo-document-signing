# Blog demo prompt

/architect:plan

I’m planning a new multi-tenant B2B document-signing product. Team is 5 engineers and the goal is a credible MVP in 10 weeks. Make reasonable assumptions where needed and capture open questions explicitly rather than stopping for clarification unless something is genuinely blocking.

Requirements:
- sender/admin experience for creating documents, templates, recipients, and tracking status
- public signer flow entered from email magic links
- webhook callbacks into customer systems when document/signature state changes
- immutable audit trail for every signature event and important document transition
- admin analytics/dashboard for tenant admins and internal support
- tenant isolation, RBAC, and reasonable security defaults for a B2B product
- pragmatic reliability for async work like notifications, webhooks, and audit/event processing
- clear ownership of source-of-truth data and clear boundaries between sync request paths and background work

Produce a concrete architecture recommendation with runtime boundaries, data ownership, key sync vs async flows, trust boundaries, and the main design decisions we should lock before implementation. For the major tradeoffs, recommend a default rather than just listing options.