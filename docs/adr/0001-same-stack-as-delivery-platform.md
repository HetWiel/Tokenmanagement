# Same stack as the Delivery Platform: TypeScript (Next.js + Postgres) on Azure West Europe

Token Management is built in TypeScript with Next.js and PostgreSQL. It runs as one deployment in the Provider's Azure subscription, in the West Europe region. This matches the Delivery Platform (its ADR 0003) rather than the ITAM Agent's Python stack, because the two tools are expected to be integrated later, and one language makes it cheap to share code, types and people between them. West Europe keeps all Client data in the EU.
