# Mail Best Practices

## Choose Queued or Synchronous Delivery

ShouldQueue makes mail dispatch queued by default. Use it when durable asynchronous delivery fits the workflow; preserve intentional synchronous delivery and error/timing contracts rather than converting all mail automatically.

## Preserve Transaction Timing for Queued Mail

For queued mail that must observe committed state, use afterCommit() or the existing connection configuration. Confirm transaction boundaries and existing timing before changing them; the mechanism does not provide atomic delivery with database writes.

## Use `assertQueued()` Not `assertSent()` for Queued Mailables

Use Mail::fake() and assertQueued() for a queued mailable; use assertSent() for the synchronous path. A mailable without ShouldQueue can still be explicitly queued, so assert the workflow actually exercised rather than its class declaration alone. These assertions establish fake dispatch, not worker execution or delivery.

## Choose Content Tools to Fit Existing Templates

Markdown mailables can produce HTML and plain-text versions and reuse existing components. Existing Blade/HTML/text templates are also valid; do not migrate them merely for preference. Use the installed generator's supported --markdown option only when creating an authorized new mailable. Preserve escaping, recipient authorization, locale and existing public content contracts.

## Distinguish Content and Dispatch Assertions

Use direct mailable rendering/content assertions for HTML/text content and Mail::fake() with assertSent()/assertQueued() for dispatch, recipients and contract-relevant data. Keep the responsibilities explicit; a focused test can cover both when that improves the regression signal. Rendering may still trigger application queries or attachment reads, so inspect and isolate those dependencies. No real mail delivery without authorization.

Source: Laravel 12.x [mail](https://laravel.com/docs/12.x/mail), especially [queueing](https://laravel.com/docs/12.x/mail#queueing-mail) and [testing](https://laravel.com/docs/12.x/mail#testing-mailables). Match installed APIs, transport and transaction configuration.
