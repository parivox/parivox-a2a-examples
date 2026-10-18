# Contributing

Each agent under `agents/` is self-contained. To work on one:

```bash
cd agents/research-specialist-sdk
python -m venv .venv && .venv/bin/pip install -r requirements-dev.txt
.venv/bin/python -m pytest -q
cd ../.. && agents/research-specialist-sdk/.venv/bin/ruff check .   # lint runs from the repo root, as in CI
```

CI also runs the official [`a2a-tck`](https://github.com/a2aproject/a2a-tck) against the
built image (see `.github/workflows/ci.yml`), and `gitleaks` over the full history.

- Configuration comes from environment variables only. Copy `.env.example` to `.env` for
  local runs; `.env` is gitignored, and should stay that way.
- Never commit credentials, account identifiers, ARNs or internal hostnames. See
  [SECURITY.md](SECURITY.md).
- Keep the agents independent: no shared library between them, so a client that works
  against both is exercising the protocol.
