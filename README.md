# Myspa ERP System — API Documentation

Public developer documentation for integrating with the Spa Management SaaS API.

## Available APIs

| API | Auth | Use it for |
|-----|------|------------|
| [Spa Management REST API](docs/rest-api.md) | Bearer token | Full API — login, business switching, customers, and consultation records. Build mobile apps, integrations, and internal tools. |
| [Lead Capture API](docs/lead-capture-api.md) | Business API key (`X-Api-Key`) | Push leads into the CRM from external sources (website forms, landing pages, funnels). |

**Which one do I need?**

- Building an app or integration that reads spa data on behalf of a logged-in user → **Spa Management REST API**.
- Just sending leads from a website form or funnel, no user login → **Lead Capture API**.

## About

This repository hosts the public-facing API reference for developers and integration partners.

## Support

Questions about API access or your API key? Contact your account admin or support at [support@myspa.co.ke](mailto:support@myspa.co.ke).

## License

This documentation is provided under the [MIT License](LICENSE).
