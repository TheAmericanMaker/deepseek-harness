# Closeout — architecture

## Summary

- DeepSeek Harness is a plugin-based agent harness on vendored Cordis ("everything is a plugin").
- 233 npm packages in packages/<group>/<pkg>, plus vendor/ (9 vendored framework packages), apps/ (cli + web), python/ (SDK + runtime), native/ (Landlock launcher), examples/, website/.
- Dependency direction is acyclic: vendored framework -> zero-dep util -> core spine -> capability seams -> bundles/shells.
- Public surfaces: dsh CLI, Web UI, ACP server, JSON-RPC SDK, Python SDK, native launcher, hook bridges, MCP client.
- Concurrency: async/await event loop + worker threads + subprocess/PTY providers + OS-specific sandbox backends.
- Build: pnpm + tsc project references + tsdown + vite; tag-driven release (v* npm, python-v* wheels).
