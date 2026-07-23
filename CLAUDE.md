# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

```bash
# Install dependencies
pipenv install

# Run the Flask application (http://localhost:5000)
pipenv run python app.py

# Format / lint
pipenv run black .
pipenv run isort .
pipenv run pylint [module_name]

# Tests (no test files exist yet)
pipenv run pytest .
```

## Architecture

The core pipeline in `ice_breaker.py::ice_break_with(name)` runs in sequence:

1. **Agent** (`agents/linkedin_lookup_agent.py`) — uses Tavily web search to find a LinkedIn profile URL for the given name
2. **Scraper** (`third_parties/linkedin.py`) — calls Scrapin.io API to fetch structured profile data
3. **Agent** (`agents/twitter_lookup_agent.py`) — finds Twitter username (currently `ice_breaker.py` calls the mock version directly)
4. **Three LangChain chains** (`chains/custom_chains.py`) run against the scraped data and return Pydantic-validated objects via `output_parsers.py`: `Summary`, `TopicOfInterest`, `IceBreaker`
5. **Flask** (`app.py`) serializes those objects to JSON via their `.to_dict()` methods

### LLM configuration

- Agents use **GPT-4o-mini** (temperature=0)
- Summary/interests chains use **GPT-3.5-turbo** (temperature=0)
- Ice breaker chain uses **GPT-3.5-turbo** (temperature=1) for creativity

### Mock / development mode

Both third-party scrapers have offline fallbacks:
- `scrape_linkedin_profile(url, mock=True)` — fetches a hardcoded Gist instead of hitting Scrapin.io
- `scrape_user_tweets_mock(username)` — returns static tweet data; currently hardcoded in `ice_breaker.py`

To test without API credits, pass `mock=True` to the LinkedIn scraper and the Twitter call is already mocked.

## Environment Variables

Copy `.env.example` to `.env`. Note: `.env.example` still contains the legacy `PROXYCURL_API_KEY` key — the code actually reads `SCRAPIN_API_KEY`.

Required:
- `OPENAI_API_KEY`
- `SCRAPIN_API_KEY` — Scrapin.io for LinkedIn data
- `TAVILY_API_KEY` — web search for profile discovery

Optional Twitter API (currently bypassed by mock):
- `TWITTER_API_KEY`, `TWITTER_API_KEY_SECRET`, `TWITTER_BEARER_TOKEN`, `TWITTER_ACCESS_TOKEN`, `TWITTER_ACCESS_TOKEN_SECRET`

Optional LangSmith tracing (if `LANGCHAIN_TRACING_V2=true`, a valid `LANGCHAIN_API_KEY` is required or the app errors):
- `LANGCHAIN_TRACING_V2`, `LANGCHAIN_API_KEY`, `LANGCHAIN_PROJECT`
