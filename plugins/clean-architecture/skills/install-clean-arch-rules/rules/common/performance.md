# Performance Optimization

## Model Selection Strategy

**Haiku 4.5** (90% of Sonnet capability, 3x cost savings):
- Lightweight agents with frequent invocation
- Pair programming and code generation
- Worker agents in multi-agent systems

**Sonnet 4.6** (Best coding model):
- Main development work
- Orchestrating multi-agent workflows
- Complex coding tasks

**Opus 4.5** (Deepest reasoning):
- Complex architectural decisions
- Maximum reasoning requirements
- Research and analysis tasks

## Context Window Management

Avoid last 20% of context window for:
- Large-scale refactoring
- Feature implementation spanning multiple files
- Debugging complex interactions

Lower context sensitivity tasks:
- Single-file edits
- Independent utility creation
- Documentation updates
- Simple bug fixes

## Context Hygiene

Monitor Claude Code context usage to maintain optimal performance:

**Context thresholds**:
- **70% context**: Ideal working range - continue normally
- **90% context**: Start freeing up context:
  - Close unused files
  - Clear unnecessary chat history
  - Break large tasks into smaller sessions
- **>95% context**: STOP and clean up immediately:
  - Save work in progress
  - Start fresh session
  - Plan task breakdown

**Best practices**:
- Review open files regularly and close unused ones
- Keep chat focused on current task
- For large refactorings, plan multiple sessions at the start
- Save intermediate progress before context reaches 90%
- Use clear, focused prompts to minimize back-and-forth

Large refactorings spanning many files should be planned as multi-session work from the beginning.

## Extended Thinking + Plan Mode

Extended thinking is enabled by default, reserving up to 31,999 tokens for internal reasoning.

Control extended thinking via:
- **Toggle**: Option+T (macOS) / Alt+T (Windows/Linux)
- **Config**: Set `alwaysThinkingEnabled` in `~/.claude/settings.json`
- **Budget cap**: `export MAX_THINKING_TOKENS=10000`
- **Verbose mode**: Ctrl+O to see thinking output

For complex tasks requiring deep reasoning:
1. Ensure extended thinking is enabled (on by default)
2. Enable **Plan Mode** for structured approach
3. Use multiple critique rounds for thorough analysis
4. Use split role sub-agents for diverse perspectives

## Build Troubleshooting

If build fails:
1. Use **build-error-resolver** agent
2. Analyze error messages
3. Fix incrementally
4. Verify after each fix
