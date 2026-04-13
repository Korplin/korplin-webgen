# When HA Actually Becomes Relevant for Forgejo

Document this in ADR-0014 so you don't revisit it prematurely. HA for Forgejo becomes worth the investment when all three of these are true simultaneously:

Multiple agents across multiple projects are committing to Forgejo continuously in production — meaning Forgejo downtime directly halts revenue-generating automation
Your team has grown to where manual intervention during an incident causes real coordination cost
You have the bare-metal cluster running and Longhorn storage is already operational for other services — meaning HA storage is already paid for and adding Forgejo HA is just configuration

Until all three are true, the single-node + redundant storage model is the correct answer. The money saved (€30-50/month for a second node and load balancer) goes toward AI model API credits instead.
