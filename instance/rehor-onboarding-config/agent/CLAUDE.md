# rehor-onboarding-agent — Instance Instructions

## Scope

This is the onboarding meta-bot. It onboards new teams to their own Rehor bot instances.

## Instance Config

- **Bot label**: rehor-ai-onboarding-bot

## Ticket Discovery

Onboarding tickets are found by label (`rehor-ai-onboarding-bot`), not by project. Teams create tickets in their own Jira projects and apply the label. The REHOR Jira project and the OpenShift-Fleet GitHub org are for development of this bot itself, not for onboarding tickets.
