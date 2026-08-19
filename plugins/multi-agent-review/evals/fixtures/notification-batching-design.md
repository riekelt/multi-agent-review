# Notification batching, design

Date: 2026-08-19
Status: Draft (eval fixture: contains deliberately planted defects; see evals.json)

## Summary

Notifications currently send one push per event. This design batches them per user into a digest to cut push volume.

## Behavior

1. Events append to a per-user queue in `NotificationQueue`.
2. A scheduler flushes each queue every TBD minutes.
3. The flusher renders the digest with `DigestTemplateVNext` and sends it through the provider.
4. Errors during the flush are handled appropriately.

## Rollout

Ship to all users at once.
