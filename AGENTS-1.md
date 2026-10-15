I'll check the current Copilot custom-agent setup for IntelliJ, then build the agent files for you.

One honest note first: no AI agent can *guarantee* defect-free code. What I built into the agent instead is the next best thing — it must compile and run your tests after every change, fix failures, and only report success on a green build.

**Step-by-step in IntelliJ**

1. **Update IntelliJ** to a recent version (Help → Check for Updates).
2. **Install/update the plugin:** Settings → Plugins → Marketplace → search "GitHub Copilot" → Install (or Update) → restart IDE. Custom agents, sub-agents, and plan agent are now generally available in GitHub Copilot for JetBrains IDEs, so you need a current plugin.
3. **Sign in:** click the Copilot icon in the status bar → Login to GitHub → finish in the browser.
4. **Create the folder** in your project root: right-click project → New → Directory → `.github/agents`.
5. **Add the agent file:** copy `jsf-spring-expert.agent.md` into `.github/agents/`. (Alternatively, open Copilot Chat → agents dropdown → Configure Custom Agents → create new, and paste the contents.)
6. **Optional – use it in all projects:** you can define custom agents at the global level using the .agent.md file under ~/.copilot/agents, making them available across all your workspaces.
7. **Allow terminal commands:** the agent needs to run `mvn compile` and `mvn test`. Approve them when prompted, or set auto-approve under Settings > GitHub Copilot > Chat > MCP Server and Tool Auto-approve Configuration.
8. **Make sure Maven/Gradle and JDK 17+ work** from IntelliJ's terminal (`mvn -v`), otherwise the verification step can't run.
9. **Use it:** open Copilot Chat → switch to **Agent** mode → pick **JSF Spring Expert** from the agent dropdown → give a task, e.g. *"PLAN: migrate the orders module from javax/JSF 2.3 to Jakarta Faces 4."*
10. **Commit `.github/agents/`** to Git so your whole team gets the same agent.

**Tips for best results**
- Start big jobs in PLAN mode, review the plan, then say "proceed".
- Have real unit tests — the agent's "error-free" check is only as strong as your test suite.
- Pick a strong model in the chat model selector for migration work.

Sources:
- [GitHub Changelog – JetBrains agentic GA (Mar 2026)](https://github.blog/changelog/2026-03-11-major-agentic-capabilities-improvements-in-github-copilot-for-jetbrains-ides/)
- [GitHub Changelog – global .agent.md in JetBrains (May 2026)](https://github.blog/changelog/2026-05-13-introducing-copilot-cli-agent-and-unified-sessions-view-in-github-copilot-for-jetbrains-ides/)
- [GitHub Docs – Create custom agents](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/create-custom-agents)