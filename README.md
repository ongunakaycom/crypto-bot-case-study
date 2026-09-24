# Crypto Bot — AI-Powered Crypto Analyst (Case Study)

> **Source code is private.** This repository documents the architecture, 
> API contract and engineering decisions behind a production-deployed 
> AI crypto analyst. It is a case study, not the application source.

🔗 **Live demo:** [crypto-bot-frontend-lhne.vercel.app](https://crypto-bot-frontend-lhne.vercel.app)

---

## 🎯 What it does

An AI crypto analyst that answers natural-language questions about Bitcoin
using real market data. It does not hallucinate numbers — it computes 
indicators (RSI, trend, support/resistance) from 90 days of historical 
price data, then asks Gemini to explain those numbers in plain language.

---

## 🏗️ Architecture

[Buraya mermaid diyagramı gelecek]

---

## 🔌 API Contract

See [`api-contract.md`](./api-contract.md).

---

## 🧠 Engineering Notes

See [`engineering-notes.md`](./engineering-notes.md).

---

## 📊 System Maturity

| Layer | Status |
|-------|--------|
| Auth | ✅ Production |
| Chat | ✅ Production |
| Profile | ✅ Production |
| Deployment | ✅ Production |
| Security | 🟡 Hardening in progress |
| Observability | 🟡 Partial |
| Testing | 🔴 Roadmap |
| Docs | 🟡 In progress |

---

## 📜 License

MIT — see [LICENSE](./LICENSE).