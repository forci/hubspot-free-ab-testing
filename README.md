# HubSpot Free A/B Testing

A/B test your HubSpot email campaigns — no Marketing Hub Professional required.

HubSpot Free and Starter plans don't include native A/B testing. This skill gives you the same workflow: split your contacts into randomized groups, send different email variants to each, measure results, and send the winner to everyone else.

Works with any number of variants (A/B, A/B/C, ...) and any contact list size.

## What it does

1. Takes your HubSpot contact export (CSV) or pulls contacts via API
2. Filters for eligible marketing contacts (removes unsubscribed, non-marketing, duplicates)
3. Randomizes contacts into equal groups — one per email variant
4. Generates import-ready CSV files for HubSpot static segments
5. Walks you through importing, sending, evaluating results, and sending the winner

## Install

This is a [Claude Code skill](https://skills.sh). Install it with:

```bash
npx skills add forci/hubspot-free-ab-testing
```

Then ask Claude: _"I want to A/B test an email campaign in HubSpot"_

Also works with Cursor, Windsurf, Codex, and other AI coding agents.

## Requirements

- HubSpot account (Free or Starter) with contacts
- Email variants already drafted in HubSpot
- Python 3 (for contact filtering — script is generated on-the-fly, nothing to install)

## License

MIT — see [LICENSE](LICENSE)
