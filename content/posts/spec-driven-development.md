---
title: "Spec Driven Development: Getting Better Code from AI Assistants"
date: 2025-11-16T20:20:07+01:00
draft: false
categories: ["AI and developer tools"]
---

LLMs like Claude, Cursor, and GitHub Copilot are getting better at solving coding problems. But anyone who has ever written a business feature knows that the real challenge is not just writing the code. The code should follow your existing coding patterns, respect organization specific linting rules, use the right libraries, and fit into your architecture. So no matter how tempting it is for a developer to just copy paste the title and maybe description from the ticket in your favourite ticket management system its highly likely not going to produce code which is closer to the final version.

What LLMs are lacking is context - the patterns you follow, the libraries you use, the architectural decisions your team has made. This is where Spec Driven Development comes into play.

Now let me be clear upfront - even with the best specs, LLMs can still produce code that needs fixing. This is not a silver bullet. But it does significantly improve the starting point.

Although there’s no formal definition of Spec Driven Development, it often means coming up with a structured approach to provide as detailed and as much contextual information as possible to the LLM. Think of it as giving the LLM a proper briefing before sending it to do the work. The more context you provide upfront, the better the output.

So how do you actually do it? What should a spec look like? I’ll start with a very basic approach and move towards more structured approaches. In my daily work I use Claude Code and Cursor so my examples will focus on them.

If you want the short version: simple markdown files work for most tasks, Cursor has a built-in plan mode that’s nice, and for large projects there’s GitHub Spec Kit. Pick based on complexity.

## 1. Simple Markdown Specs

If its an incremental feature or a bug fix, a quick start approach can be a simple markdown file. This works surprisingly well for small to medium sized tasks. You don’t need anything fancy.

Create a spec file with sections like:

**Use Case** - What are we trying to achieve? What’s the business problem?

**Tech Spec** - How should it work technically? What endpoints, what data models, what flow?

**Output** - What’s the expected output or behavior?

You can also add sections like **Coding Style** or **Guidelines** where you mention anything org specific - maybe you always use a certain error handling pattern, or you have specific logging requirements.

If you’re using Claude Code or Cursor these specs can even go into your project level or global level configuration files like `.cursorrules`, `.agent.md`, or `claude.md` files. This is nice because then every conversation automatically gets that context.

Once you have your spec ready, you can use a simple prompt:

    Follow the specification at <your-spec-file>, think step by step, plan the tasks, show the tasks and wait for approval before start implementing.

This approach works great when:

- You’re working on incremental features
- The task is well-defined
- You already know what you want

## 2. Cursor Plan Mode

In Cursor I follow a similar process but use the Plan mode. It generates a plan with a list of tasks and steps through each of them. Once done you can review all the changes in one go.

The nice thing about plan mode is it forces you to think through the entire task upfront. The LLM generates a plan, you approve it, and then it goes through task by task making changes. You get to see the plan before any code is written which gives you a chance to course correct early.

Quality wise I don’t see much difference between this and the simple readme approach and honestly I have no strong preference among them. Sometimes I use plan mode, sometimes I just use a spec file - it depends on my mood more than anything else. Both work well for incremental features and bug fixes.

The key is having that spec or outline upfront. Without it you’re basically asking the LLM to guess what you want and that rarely ends well.

## 3. GitHub Spec Kit

The above two approaches are particularly good for incremental features and bug fixes, but if you’re solving a relatively large use case or a greenfield coding task you need a more planned approach. For these kind of scenarios I’ve been trying out GitHub Spec Kit. It takes a much more formal structured approach.

> Note: I’m still early in my exploration of Spec Kit so this section is based on limited hands-on experience. Your mileage may vary.

The pillars of Spec Kit are:

**Constitution** - This is where you define your coding style, org lint style, architectural principles - basically stuff that cuts across specific tasks. Think of this as your team’s coding philosophy. Things like “we always use dependency injection” or “error messages should be user friendly” go here.

**Specification** - This is where you specify what and why. What feature are we building? Why are we building it? What problem does it solve? This section forces you to think through the requirements properly before jumping into code.

**Plan** - Here you mention your tech stack, architecture, high level design. Are we using Spring Boot? What database? REST or GraphQL? Microservices or monolith? All the architectural decisions go here.

**Tasks** - Based on the above, Spec Kit generates a detailed task list. Each task is specific and actionable.

**Implement** - Finally you go through and implement each task.

From what I’ve seen, Spec Kit has features to help gather additional context and documentation which can be helpful for bringing in current best practices. Though I’m still early in exploring this tool so take my observations with a grain of salt.

### Pros

- More formal and structured approach
- Before coding a large feature it makes sense to do a lot of upfront specification
- LLM has way more context which otherwise you’d have to provide manually and most likely you would have skipped some important details
- Helps bring in current documentation and best practices
- Good for greenfield projects where you’re starting from scratch
- Forces you to think through architecture and design decisions upfront

### Cons

- Takes longer time - the whole process is more involved
- Generates multiple files - you’ll have constitution files, spec files, plan files, task files which can clutter your repo if you’re not careful about organization
- For small tasks its definitely overkill - if you’re just fixing a bug you don’t need all this ceremony
- Can feel a bit heavyweight for quick iterations
- Learning curve - figuring out the right level of detail for each section takes practice

## When to use what?

I think of it like this:

**Small tasks, bug fixes, quick features** - Simple markdown specs or Cursor plan mode. Just write down what you want, give it to the LLM, done.

**Medium features** - Still markdown or plan mode but spend a bit more time on the spec. Add more context about existing patterns.

**Large features, greenfield projects** - GitHub Spec Kit or similar formal approach. The upfront time pays off because the LLM has proper context and makes better decisions.

The common thread across all these approaches is context. The more context you give the LLM about your codebase, your patterns, your requirements - the better the output. Spec Driven Development is really just about being intentional with that context instead of hoping the LLM figures it out.

In my experience, taking time to write a proper spec often saves back and forth with the LLM trying to course correct. But there’s a learning curve to figuring out what level of detail works best and honestly sometimes for truly trivial changes its just faster to write the code yourself.

## Important caveats

A few things to keep in mind:

**You still need to review the code** - Specs don’t magically make LLM-generated code perfect. You still need to understand what the code does, test it properly, and review it like you would any other code.

**Specs can be wrong too** - If your spec has incorrect assumptions or missing requirements, the generated code will reflect those problems. Garbage in, garbage out still applies.

**Not everything needs a spec** - For genuinely trivial changes like updating a string or fixing a typo, writing a spec is just overhead. Use your judgment.

**This is still evolving** - AI coding assistants are improving rapidly. What works today might be different in six months. Stay flexible and keep experimenting with what works for your team.
