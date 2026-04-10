# About Project:

Project name: Corepixen website generation automation

We are web design studio. I am learning and can say that I am at beginner level of DevOps, GitOps and AI Engineering. But feeling myself confident in that because I was coding previously. I am working in Cursor and using mostly agentic coding.

In our business level we are making for potential costumer free drafts of their possible future websites before signing contract. We want **maximally expand** that idea and AI automate every possible step.

By maximally expand we are thinking of new business and marketing model for maximal amount of lead generation and sales.

## Pipeline description

We are starting to develop this pipeline:

1. We have agent tier 1 who is constantly searching and scanning web for potential costumers. Maybe they don't have website, maybe site is outdated, maybe it is fresh company and they didn't made to their website yet, etc.
2. If tier 1 finds potential customer then it gives prompt to tier 2 who will dig up all the relevant information about client from public sources that could be useful for website creation.
3. Tier 2 will give prompt to tier 3 who is responsible for creating prompt for agent who will be "physically" building ready website based on prompt from tier 3
4. Tier 3 gives prompt to tier 4 that website is ready. Tier 4 evaluates site for bugs, design issues, broken links, language and so on. If it finds something that must be corrected then tier 4 sends prompt back to tier 3 for fixing. If site is ready for human review then tier 4 will send prompt to tier 5 that website is ready with all relevant additional information.
5. Tier 5 will update our project management app that site x is ready for human review.
6. Human will review site and make prompt for corrections that will go to tier 3 and loop activates again until site is ready for being published, but no publishing yet.
7. Human makes last review, fills out necessary fields in PM app and changes state to "To be published"
*agents can work in loops, if necessary then they will be talking to each other if it gives better results.
** As drafts we will be building just html, css and javascript based websites.
*** Outlined pipeline is subject to change, it just brief idea discription how I imagine it in work right now.

Downstream tiers will physically publish site and write email to potential costumer about their website draft and notifies our team about that.

We will use proprietary AI models providers such as Claude, OpenAI or any other that would be helpful in pipeline. If there is any open source AI models that could help achieve our goal, then we will happily incorporate it. Also we see no problem hosting AI models ourselves for example through GPU compute service runpod.io. Although all the infrastructure must be open source, Git based and reproducible. First we will host our infrastructure on VPS server or servers, later we move infrastructure to our own bare-metal servers that will form Docker, Podman, K3s or K8s clusters for high availability. Stack is not hardcoded yet and this is something we have to discuss.

Complexity is not problem since I am beginner DevOps AI Engineer. But we don't want to pay for automation itself. We will pay only AI models searching web, executing prompts, physically building websites and sending mails. Complexity is not problem. 

Solution must be GitOps reproducible and work in isolated environment. What are your suggestions on stack, tooling, workflows and pipeline?

# Project goal:

To have fully functional owned SaaS style deployment where agents finding lead, generating autonomically draft websites, communicating through e-mails with team and potential clients, etc. Goal will be clearly defined later, this is just basis and will be expanded later.

# Tasks:

How to intrpret task statuses: - [X] = done. [IP] = In Progress. That is or are (if multiple selected) tasks that we solving right now in this thread. - [ ] = Planned


## Task 1
- [IP] Define and outline whole stack that is necessary for that Corepixen website generation automation project. On chosen stack depends further Git repository structure in Task 2 and later files contents.

## Task 2
- [ ] Create Git repository directory structure

## Task 3
- [ ] Create and write contents of every necessary file for successful deoployment

## Task 4
- [ ] N/A

## Task 5
- [ ] N/A

## Task 6
- [ ] N/A

# ADRs

ADRs in this file are very compact. Just name and brief description if necessary. Separate folder with with correct ADR files exists in this repo.

# ADR-0001-linux-based-systems-for-deployment

# ADR-0002-git-based-single-source-of truth

# ADR-0003-deployed-and-managed-gitops-way

# ADR-0004-reproducible-build-process

# ADR-0005-first-vps-later-in-house-bare-metal

# ADR-0006-open-source-infrastructure-and-tooling

Exception: is AI tooling. We will use any paid proprietary AI model for achieving results faster and bettr. All paid and proprietary AI models and solutions are allowed.





