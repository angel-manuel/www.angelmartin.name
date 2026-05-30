---
layout: post
title: "Why did I build an auth gateway for AI Agents"
post_ref: auth_gateway_for_ai_agents
lang: en
categories:
tags:
  - ai
  - ai agents
  - security
  - authorization
  - overslash
---

As many others recently I got myself interested in AI Agents(I feel so deep that I'm coding my own AI agent but more on that on a future post). Naturally, I spun up my OpenClaw on a VPS and started to attempt to use it at every occasion. And yet, I haven't managed to save much time on my daily life.

For many things I find that my AI can do 50% on the work. For example, I wanted to organize a BBQ event and I needed to buy meat. My AI agent was able to:

+ find a store near my location
+ Get the web page
+ see that they offered 8 different packs of products for BBQs(it did have to load and understand images for this)
+ Estimate how much food people eat
+ Suggest "menu 2 + menu 8"

The store has their phone number published, and they take orders on WhatsApp. But my agent doesn't have WhatsApp access.

Why doesn't it have access? Well, it is not out of lack of capability. There are some libraries and even a [new](https://x.com/steipete/status/1999499414419734831) [CLI](https://github.com/steipete/wacli). It involves a bit of hacking around WhatsApp linked devices undocumented API but it works. The end result is the agent can read and send messages impersonating the user.

The problem is, I don't want to give the agent direct access to my WhatsApp, I want to review the messages the agent sends before approving them.

So I started thinking about making my own authorization/approval system for AI Agents.

## How to make good approvals?

A great approval system for AI Agents would have the following:
+ **Human Readable Approvals**: plain language descriptions and context, no bash, no inline script, no HTTP/JSON
+ **Around the Agent**: the agent shouldn't be able to bypass it or "forget" to use it
+ **Flexible**: eg. don't re-prompt me for every email to this recipient
+ **Auditable**: what was done, when, who approved it...
+ **General**: Work the same for all services and actions
+ **Non blocking**: the agent doesn't stop until you approve, it can try other actions or keep working on other things until the approval arrives
+ **Delegable**: if I can approve my agent actions, can my agent selectively approve subagent actions? 

Different current attempts fail at one or more of these:
+ Coding Agent approvals block execution, are not human readable(big inline bash is hard to review even for coders).
+ Auto mode for Claude Code helps 90% of the time, and provides non-blocking by directly refusing approvals sometimes(sometimes leaving the agent unable to complete its task). It is "Around the Agent" but still based on LLM, ie. not infinitely reliable.
+ OpenClaw directly doesn't bother. Critically the Agent can access the secrets needed to perform on most normal setups. And will get them into its context(if not intermediately it will eventually)
+ IETF, OAuth, and others are coming with an avalanche of specification drafts: I've read some of them, skimmed others. While there is value in there I don't think half of the stuff will be needed.
+ Still others offer closed, proprietary solutions.

## My attempt at solving this

What if the agent had a tool named `curl` that does call the normal curl underneath BUT it also is able to inject some secrets, such as API Keys.

We can also create another tool `request_secret` that prompts the user for secrets when needed, it a way so that the secret doesn't end up in LLM context but instead goes to the harness or `curl` tool.

By doing this carefully we can **separate the Agent from the secrets**.

Then we can expand this by gating the `curl` tool.
If the agent wants to `curl("POST" "api.com", auth_header="secret_api_key")` then it needs:
+ the permission to `POST` to `api.com`: `"curl:POST:api.com"`
+ and the permission to use the `secret_api_key` secret on `api.com` :  `curl:secret:api.com:secret_api_key`
	+ it's import to limit the use of secrets to a host, or even bind each secret to a host, otherwise the Agent can exflltrate or get the secret contents by using an echo webserver

With this we now have an authorization layer **Around the Agent** and by adding logging we can make it **Auditable**. I implementing a version of this for a time on my AI Agent, it is an improvement. Anthropic has released something similar but different with Vaults on Managed Agents.

It will also work for any API key REST API but the approvals wont be human readable(normal users don't care about HTTP requests and the meat of many actions is on the HTTP payload) nor flexible(cant approve only specific actions on an endpoint).

Now, suppose we authored a YAML describing an API like this(looks and is OpenAPI but with a couple of custom keys...):

```yaml
openapi: 3.1.0
info:
  title: Google Calendar
servers:
  - url: https://www.googleapis.com
components:
  securitySchemes:
    oauth:
      type: oauth2
      x-overslash-provider: google
      # ...
paths:
  /calendar/v3/calendars/{calendarId}/events/{eventId}: 
    parameters:
      # ...
      - name: eventId
        in: path
        required: true
        description: Event identifier
        schema:
          type: string
        x-overslash-resolve:
          get: /calendar/v3/calendars/{calendarId}/events/{eventId}
          pick: summary
    get:
      # ...
    patch:
      # ...
    delete:
      operationId: delete_event
      summary: "Delete event {eventId} on calendar {calendarId}"
      x-overslash-risk: delete
      x-overslash-scope_param: calendarId
```

Then we would know that there is an API (Google Calendar), on such domain allow some methods.
These methods operate on `eventId` operate on ugly UUID ids such as `abce6476-44b3-4e16-89f7-1add4d6986de` but luckily, the `x-overslash-resolve` key on the YAMLs tells us we can convert them into something else(in this case the name/summary of the calendar event) on `/calendar/v3/calendars/{calendarId}/events/{eventId}`.

With this we can turn a:
"Do you approve `http:DELETE:https://www.googleapis.com/calendar/v3/calendars/primary/events/abce6476-44b3-4e16-89f7-1add4d6986de` with secret `secret_gcal_oauth_key`? \[Yes]\[No]"
to
"Delete event 'Crazy Party on the Moon' on calendar 'primary'? Connection: user@gmail.com 
\[Yes]\[No]"
"\[Remember for connection user@gmail.com]
\[Remember for calendar: primary and connection user@gmail.com]"

We did it by using the OpenAPI action summary, converting IDs to readable info. And we prefilled some useful remember permission keys using the `x-overslash-scope_param` extension(we use it to tell us that part of the payload might be important for scoping permissions).

Now our approval system gained **Human Readable Approvals** and **Flexible** scoping of permissions.

We can make the system **Non Blocking** too, just let the agent try the action(or ask if the action is allowed) and surface an approval permission to the user, tell the agent the approval is pending. The Agent can poll another tool on its heartbeat or after a reasonable wait or, with a bit of control over the harness we can inject the approval/rejection whenever the user acts on it.

It the system knows that an agent is a subagent of other agent we can bubble up the request via the chain, to either a human or a parent agent. If the parent agent already has the permission we give it the ability to approve or not its subagent action, it can also choose to keep bubbling the decision up until it reaches the human. This way approvals can also be **Delegated**.

We can also also support MCP in a very similar way. OpenAPI Spec can be extended to tag MCP tools accordingly. I already had [an easy to deploy MCP server for Whatsapp](https://github.com/angel-manuel/whatsapp-mcp-docker)  and this is the header of a YAML describing its tools:

```yaml
openapi: 3.1.0
info:
  title: WhatsApp
  # ...
x-overslash-runtime: mcp
servers: []
paths: {}
x-overslash-mcp:
  auth:
    kind: bearer
  # autodiscover tells us to use tools/list to get MCP tools even if not described here
  autodiscover: true
  tools:
    - name: pairing_start
      risk: write
      description: "Open a WhatsApp pair flow and return the first event (rotating QR payload, plus a phone-link code if {phone} is given)"
      # ...

```

## Closing the Loop

I implemented all the ideas above(and some extra ones) on an app called Overslash. This tools proxies all external services tool uses and can be consumed as a single MCP!

This MCP can be consumed from Claude Code, claude.ai, Openclaw and others. Here is a demo of the full flow, ending up in sending a Whatsapp message:

<video controls muted playsinline loop preload="metadata" poster="{{ '/assets/video/overslash-whatsapp-claude-poster.jpg' | relative_url }}" width="100%" style="max-width:100%;border-radius:8px;">
  <source src="{{ '/assets/video/overslash-whatsapp-claude.mp4' | relative_url }}" type="video/mp4">
  Your browser does not support the video tag.
</video>

See the full Overslash announcement at www.overspiral.com/blog/introducing-overslash/ and start using it for free at www.overslash.com

## Overslash

I implemented the above, plus a lot more:
+ Organizations/Teams
+ Agent hierarchy
+ Dashboard for connecting and defining services
+ Audit Logs

You can host it yourself or use it for free on cloud at [overslash.com](https://www.overslash.com). Or just copy this into your OpenClaw, Claude, Codex:

```
Your Human wants to give you access to external services via Overslash.
To connect, follow the instructions at: https://www.overslash.com/SKILL.md
```

