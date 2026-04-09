# Contributing: Help Us Build the Ultimate AI Guardrails

AI models are constantly updating, and the ways they hallucinate Salesforce code change every day. 

We don't just want code contributions; we want your AI failures. If your AI coding assistant (Cursor, GitHub Copilot, Claude) generated something wildly dangerous that bypassed governor limits or compromised security, we want it documented here.

## How to Contribute an AI Failure

If you caught a dangerous AI hallucination, submit a Pull Request to our `REAL_WORLD_AI_FAILURE_AND_FIX.md` file.

**Format your submission exactly like this:**
1. **The Prompt:** What did you ask the AI to do?
2. **The AI Hallucination:** Paste the dangerous code it generated. Point out exactly why it would crash a production org.
3. **The Architect Fix:** Show the corrected, enterprise-grade code that adheres to our framework.

## Updating the Guardrails
If you find that an AI model consistently ignores one of the rules in our `.cursorrules` or `CLAUDE.md` files, submit a PR to strengthen the prompt phrasing. We need these rules to be absolute, unbendable directives.

## General Guidelines
* **No untested code.** Every pattern added must compile and execute successfully in a modern Salesforce API version (55.0+).
* **Ruthless brevity.** Do not add fluff. This is a technical manual for developers, not a blog post. Keep the constraints clear and absolute.