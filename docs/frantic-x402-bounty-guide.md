# Discovering and paying a Frantic bounty with x402

> **Publication status:** local review draft. This is an explicitly labelled dry run: no wallet was connected, no payment was signed, and no public URL has been created from this workspace.

This guide is for an agent developer who has never used [x402](https://github.com/x402-foundation/x402/blob/main/specs/x402-specification-v2.md). [Frantic](https://gofrantic.com) is a public bounty venue; its payouts settle in USDC on Base. The flow is simple: discover a paid HTTP resource, call it without payment, read the 402 Payment Required challenge, sign exactly what the server requested, retry once, and verify the sealed receipt. Never infer an amount, recipient, network, or token address from a page—use the latest challenge.

## 1. Discover Frantic in the Coinbase x402 Bazaar

The Coinbase Bazaar has a read-only semantic search endpoint. The exact discovery query used here is Frantic; the network and asset filters reduce false matches:

~~~bash
curl --fail-with-body --silent --show-error --get \
  'https://api.cdp.coinbase.com/platform/v2/x402/discovery/search' \
  --data-urlencode 'query=Frantic' \
  --data-urlencode 'network=eip155:8453' \
  --data-urlencode 'asset=usdc' \
  --data-urlencode 'limit=20' | jq .
~~~

No API key is required for this GET route. Inspect every result’s resource, accepts, and extensions; do not blindly select the first hit. The current Frantic OpenAPI advertises POST https://gofrantic.com/v1/hire with the coinbase-bazaar-v2 discovery marker. If Bazaar returns a different Frantic resource later, follow the returned resource URL instead. See the [Bazaar documentation](https://coinbase-cloud.mintlify.app/x402/bazaar) for the response fields.

## 2. Read the 402 challenge and input schema

Frantic documents a public GET on /v1/hire that returns the same x402 v2 challenge an unpaid POST receives. This lets an agent inspect the price and schema without creating or funding an intake:

~~~bash
curl --silent --show-error --include --max-time 20 \
  'https://gofrantic.com/v1/hire'
~~~

Save that output verbatim when preparing a real report. In x402 v2, PAYMENT-REQUIRED is a base64-encoded header; the response body is also useful when the server exposes its JSON envelope. For this route, the documented shape is:

~~~text
HTTP/2 402
PAYMENT-REQUIRED: <base64-encoded PaymentRequired JSON>

{
  "ok": false,
  "error": "payment_required",
  "x402Version": 2,
  "resource": { "...": "resource metadata" },
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:8453",
      "amount": "<atomic USDC amount from the live response>",
      "asset": "<USDC contract from the live response>",
      "payTo": "<recipient from the live response>",
      "maxTimeoutSeconds": 300,
      "extra": { "...": "scheme-specific data" }
    }
  ],
  "extensions": { "...": "Bazaar input/output metadata" },
  "payment_required": { "...": "Frantic quote and receipt metadata" }
}
~~~

The block above is a shape-only example, not a captured response. Treat the live accepts entry as authoritative. Check that the scheme is exact, the network is Base mainnet (eip155:8453), the asset is the advertised USDC contract, and amount is an atomic-unit string. The Bazaar extension can declare the input schema; cross-check it against [Frantic’s OpenAPI](https://gofrantic.com/openapi.json).

The current /v1/hire input requires a 24-byte random request_id (48 hexadecimal characters), title, description, deliverable, one or more acceptance_criteria strings, price_cents, vendor_identity, and vendor_contact. Generate the ID once and reuse the exact same body and ID for a retry:

~~~bash
REQUEST_ID="$(openssl rand -hex 24)"
BODY="$(jq -n --arg request_id "$REQUEST_ID" '{
  request_id: $request_id,
  title: "Example vendor intake",
  description: "A small, human-reviewable bounty draft.",
  deliverable: "A Markdown guide at a stable public URL.",
  acceptance_criteria: ["The URL loads without an account."],
  price_cents: 100,
  vendor_identity: "Example Studio",
  vendor_contact: "guide@example.invalid"
}')"
printf '%s\n' "$BODY" | jq .
~~~

## 3. Sign and retry the payment

A 402 is a challenge, not a failed payment. A signer must create an x402 v2 PaymentPayload containing the selected accepts object, a signature, and the authorization fields (from, to, value, validity window, and nonce). The payload is sent in the PAYMENT-SIGNATURE header, along with the unchanged request body; any declared extension data must be echoed as required by the challenge.

With a Coinbase Agentic Wallet, the documented CLI handles the initial request, signing, and retry. Run this payment command only once, and keep its JSON output for the receipt step:

~~~bash
RESULT="$(npx awal@latest x402 pay https://gofrantic.com/v1/hire \
  -X POST \
  -d "$BODY" \
  --max-amount 3000000 \
  --json)"
printf '%s\n' "$RESULT" | tee frantic-result.json | jq .
~~~

max-amount is an approval ceiling in atomic USDC units, not a guessed price; set it to a limit you explicitly approve. This command requires an authenticated wallet with enough Base USDC. It was **not run for this draft**. Do not put a private key in the body, shell history, or guide.

## 4. Confirm the sealed receipt

Keep the successful JSON response and its receipt reference. Run the following in the same shell after step 3; it does not send another payment. Frantic exposes a public receipt index and ledger. Extract a returned receipt_ref (or receipt_id if the response uses that spelling), then search both public projections:

~~~bash
RECEIPT_REF="$(printf '%s\n' "$RESULT" |
  jq -r '.. | objects | (.receipt_ref? // .receipt_id? // empty) | strings' |
  head -n 1)"
test -n "$RECEIPT_REF" || { echo "No receipt reference returned" >&2; exit 1; }
printf 'receipt_ref=%s\n' "$RECEIPT_REF"
curl --fail-with-body --silent --show-error \
  'https://gofrantic.com/v1/receipts?limit=100' |
  jq -e --arg ref "$RECEIPT_REF" \
    '.. | objects | select((.receipt_ref? == $ref) or (.receipt_id? == $ref))'
curl --fail-with-body --silent --show-error \
  'https://gofrantic.com/v1/ledger?limit=100' |
  jq -e --arg ref "$RECEIPT_REF" \
    '.. | objects | select((.receipt_ref? == $ref) or (.receipt_id? == $ref))'
~~~

Only call the payment confirmed when the same receipt reference appears in a public response with a settled/integrity result. A transaction hash or local signer log alone is not the sealed receipt. The individual public endpoint is https://gofrantic.com/v1/receipts/{code} when Frantic returns a bare receipt code; use a returned receipt_url verbatim when one is provided.

### Dry-run disclosure

This draft intentionally contains no real PAYMENT-REQUIRED blob, signed payload, transaction, or receipt ID: the live API was not reachable from this review environment, and the requested safety boundary forbids signing or moving funds. Before publishing, capture and paste the literal discovery JSON, literal 402 headers/body, successful JSON, and the public receipt verification response. If the dry-run label is retained, state exactly what differs: a funded wallet signs the live challenge, Frantic settles the retry on Base, and the success response supplies a receipt reference that must be verified publicly.

Further reading: the [x402 v2 specification](https://github.com/x402-foundation/x402/blob/main/specs/x402-specification-v2.md) and Coinbase’s [x402 payment CLI guide](https://docs.cdp.coinbase.com/agentic-wallet/cli/skills/pay-for-service).
