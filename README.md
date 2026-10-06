# E-Commerce API Tests: Postman and Newman

Postman collections for the REST API of a React, Express and PostgreSQL online shop, run from the command line with Newman and in GitHub Actions. They check functional behaviour, authorisation, input validation and data integrity, and they can be imported straight into the Postman app for exploratory work.

The API is the same one tested in the [Playwright suite](https://github.com/mirzamaazbaig/Ecom). This repository shows the same checks the way a Postman and Newman team would write them.

| | Count |
|---|---|
| Requests in the main collection | 61 |
| Assertions in the main collection | 111 |
| Data-driven registration cases | 8 (16 assertions) |
| Run time | about 2 seconds |

## What is covered

| Folder | Checks |
|---|---|
| 0. Reachability | API root answers, response time |
| 1. Access control: anonymous | Every protected endpoint (including admin-only ones) answers 401 without a session |
| 2. Authentication | Register, session starts, duplicate email, logout ends the session, wrong password and unknown email give the same answer (no user enumeration), login |
| 3. Products | Schema of every product, limit/offset, category and price filters, sorting, case-insensitive search, SQL metacharacters in search, unsupported `sort_by`, get by id, 404, invalid ids |
| 4. Wishlist | Starts empty, add, idempotent add, listing, remove, unknown product (404), invalid id (400) |
| 5. Reviews | Post, public listing with author, rating bounds, unknown product, invalid id |
| 6. Orders | Server prices the order (the request lies about the price), stock decreases by exactly the quantity, order history, insufficient stock (409), unknown product, invalid quantities, empty cart, rejected orders change nothing |
| 7. Access control: customer | A signed-in customer gets 403 on admin endpoints and cannot change a price |
| Registration validation (data-driven) | One request run once per row of [`data/registration-validation.csv`](data/registration-validation.csv): valid, malformed emails, empty and missing fields |

Several of these checks pin defects that were found in the application and fixed; see its [defect log](https://github.com/mirzamaazbaig/Ecom/blob/claude/optimistic-faraday-sh5c40/docs/KNOWN_DEFECTS.md). Run against the application as it was before the fixes, the main collection fails 15 assertions and the data-driven run fails 7 of 16, which is what these tests are for.

## How the collection is built

- **One cookie jar, deliberate order.** Folders run in sequence: anonymous checks first (empty cookie jar), then registration and login, then everything that needs a session. The run is a single user journey, so order matters and is documented in each folder description.
- **Fresh data every run.** A pre-request script generates a unique email, so the collection can run repeatedly against the same database.
- **Variables carry state between requests.** The products request picks a well-stocked product and stores its id, name and price; later requests use them. The orders folder reads stock before and after to prove the decrement and that rejected orders change nothing.
- **Assertions say what they mean.** Test names are sentences ("Stock went down by 2"), and error cases assert both the status and the message.
- **Schema checks.** Every product is validated against a JSON schema (`tv4`) rather than spot-checking a few fields.
- **Data-driven validation.** The second collection has one request and a CSV; a pre-request script turns each row into a body (`<unique>` makes a fresh email, `<missing>` omits the field).
- **Environments.** `environments/local.postman_environment.json` holds `baseUrl` and `rootUrl`; override with `--env-var` for other hosts.

## Running it

Prerequisite: Node.js 18+ and the application's API running (PostgreSQL plus the Express server on port 5000, see the application's README).

```bash
npm install
npm test               # main collection, HTML report in reports/api-flow.html
npm run test:data      # data-driven registration validation
npm run test:all       # both
```

Against another host:

```bash
npx newman run collections/ecommerce-api.postman_collection.json \
  -e environments/local.postman_environment.json \
  --env-var baseUrl=https://staging.example.com/api --env-var rootUrl=https://staging.example.com
```

To use the collections in the Postman app, import the two files in `collections/` and the environment file.

## Continuous integration

[`.github/workflows/api-tests.yml`](.github/workflows/api-tests.yml) runs on every push and pull request, and can be started manually with a chosen application ref:

1. Checks out this repository and the application repository.
2. Starts PostgreSQL as a service container, creates and seeds the database.
3. Starts the API and waits until it answers.
4. Runs both collections with Newman; any failing assertion fails the build.
5. Uploads the HTML reports (`newman-reports`) and, on failure, the API log.

The workflow tests the application branch that contains the fixes (`ECOM_REF`); change it to the application's default branch once the fixes are merged there.

## Limitations

- No admin success paths (create, update, delete product): promoting a user to admin needs database access, which a pure API collection does not have. The 401 and 403 boundaries are covered, and the admin flows are covered in the Playwright suite.
- No load or performance testing; the response-time check is a single sanity threshold.
- The orders folder uses up a little stock on each run. The seeded catalogue has plenty; re-seed the database if a run reports insufficient stock after many local runs.
