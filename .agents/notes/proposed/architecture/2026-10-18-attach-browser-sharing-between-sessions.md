# Agent Note: Let a profile share one attached browser between Sessions

Status: proposed

English | [中文](2026-10-18-attach-browser-sharing-between-sessions.zh.md)

## Problem

The browser-use contract states attachment ownership as an absolute: "An attached browser remains externally owned. The provider reserves it for one Session within that provider instance, preserves its existing browser state, and rejects simultaneous attachment by another Session" ([Browser operations](../../../../docs/subsystems/browser-use.md)). The decision that owns that sentence is [Browser-use provider registration](../../implemented/architecture/2026-09-12-browser-use-provider-registration.md), which repeats it as a property of the runtime it describes.

The default is sound: two Sessions driving one external browser can fight over its tabs and its logins, and a provider that admits both without saying so turns that into an invisible failure. The problem is that the contract stops there. It offers no second sentence, so a provider cannot let a profile choose the opposite trade knowingly, even though the case is ordinary: one person, one signed-in Chrome, several conversations. The Session that reached the browser first keeps the whole reservation until it closes that browser or is disposed — an idle conversation blocks the browser for as long as it stays alive — and the sanctioned ways out are to wait, or to launch a browser of one's own, which discards exactly the logins the attachment was made for.

A missing sentence has not stopped providers from shipping one. At least one out-of-tree provider offers an explicit opt-in (`attachSharing: shared` in `@ricardochen1996/dsh-browser-use`), so attachment behavior is now decided per provider while the contract still reads as a prohibition. That is the drift this note asks to settle: either refusing the second Session is a requirement of the contract, or it is a default a profile may decline — and the answer belongs where provider authors read it.

## Proposal

Keep exclusive attachment as the contract's default, and name the deliberate opt-out beside it. In [Browser operations](../../../../docs/subsystems/browser-use.md) § Session ownership, the paragraph about the attached browser gains a second sentence after the reservation, and the Chinese document gains its pair:

> An activation may explicitly opt out of that reservation for its own provider instance, in which case every live Session of that instance may drive the attached browser at the same time. Sharing must keep each Session's connection and tab its own: a Session closes only the tab it opened, and teardown never closes the external browser.

The decision record that owns the reservation — [Browser-use provider registration](../../implemented/architecture/2026-09-12-browser-use-provider-registration.md) — is updated in the same change so the two can never disagree, as [implemented/AGENTS.md](../../implemented/AGENTS.md) requires.

Two things the contract should deliberately not say. It should not name a configuration key: the key belongs to the provider's own surface, and writing it into the contract would freeze a provider's YAML. And it should not describe the mechanism: connection pooling, per-daemon tabs and reservation bookkeeping are provider mechanics, and the contract is read by authors of several of them.

### What sharing must guarantee

Shared attachment is acceptable only with these invariants, and the sentence above makes the first two normative:

- **One tab per Session.** Each Session drives its own tab over its own connection; it does not navigate, adopt, or close another Session's tab.
- **Teardown is local.** Closing a Session's browser closes that Session's tab and disconnects. It never terminates or navigates the external browser another Session is still using.
- **Shared state is intentional.** Cookies, storage and sign-in identity belong to the external browser, so sharing means two conversations act as the same person. That is what attaching is for, and it is why the choice belongs to a profile rather than to a provider's default.
- **A joiner cannot evict.** A Session that attaches to an already-shared browser takes its own tab; it does not release, reassign, or expire another Session's claim.
- **The default stays exclusive.** An activation that says nothing behaves exactly as it does today.

One known interaction belongs in the implementing provider's documentation rather than in the contract: a browser-harness-style engine may tidy away unused **blank** tabs when a Session opens its first page, which can close a tab another Session has not used yet. It is bounded to blank pages and the affected Session opens a fresh tab on its next call, but a provider that offers sharing should say so.

## Alternatives considered

- **Keep the prohibition absolute.** A second conversation launches a browser of its own. Rejected: the launched browser has none of the user's logins, which is the entire reason to attach, so the common case is served only after re-authenticating.
- **Make sharing the contract default.** Rejected: it silently changes what every existing attach activation does, and the exclusivity rationale — tabs and logins — stays real for the default. A default that is right most of the time should not be flipped to serve the case that would opt out anyway.
- **Let the shared service arbitrate.** `ctx.browserUse` could hand out a lease with several holders. Rejected for now: the service owns one provider slot, not browser resources, and the activation rules the shared runtime owns are already where "who may attach" lives. A lease there is a larger change than a contract sentence and is not needed to stop the drift.
- **Decide it in the provider's own documentation only.** Rejected: the contract sentence is normative and reads as a prohibition, so a provider author who follows it will not offer the opt-out, and the next reviewer reads the out-of-tree behavior as a defect.
- **Record nothing and leave the drift.** Rejected: two providers then answer "may a second Session attach?" differently with nothing recording which answer was intended.

## Acceptance criteria

- [Browser operations](../../../../docs/subsystems/browser-use.md) and its Chinese pair state the reservation as a default with a named opt-out, in the same terms, and the decision record that owns the reservation is updated or superseded in the same change.
- No normative sentence forbids a provider from offering the opt-out, and no sentence requires a particular key name or implementation.
- The invariants above are stated where a provider author reads them, and the default is unchanged: two live Sessions on one attached browser still end with the second refused.
- Observable verification for a provider that offers sharing: two live Sessions drive one attached browser at once, each in its own tab; closing one Session's browser leaves the other Session's tab and the external browser running; and a profile that asks for the default still refuses the second Session.
- Documentation gates pass: the note is `Status: proposed` under `proposed/architecture/`, its English and Chinese sides stay paired, and `pnpm run verify-agent-note-format` and the doc-sync gate agree.

## Risks

- **Identity mixing.** Sharing makes it easier for two conversations to act as the same signed-in person. The mitigation is that it is opt-in and the profile chose to attach; the contract must not present sharing as harmless.
- **Cross-Session interference.** A provider bug becomes user-visible as "another conversation closed my tab". The invariants are only as good as the provider implementing them, which is why they are stated normatively rather than described.
- **Races inside one browser.** Several CDP clients on one browser is supported, but same-site storage and service workers are shared; a provider must not promise an isolation the engine cannot deliver.
- **Reading permission as a default.** A later author may read "may be shared" as "should be shared". The wording must keep the default sentence first and the opt-out explicitly the profile's choice.
- **Cost of being wrong.** If sharing turns out to be harmful, the default is untouched: the rollback is a profile reverting one setting, not a contract change.

## Related

- [Browser-use provider registration](../../implemented/architecture/2026-09-12-browser-use-provider-registration.md) — owns the reservation sentence this proposal qualifies.
- [Browser operations](../../../../docs/subsystems/browser-use.md) — the contract document to amend.
- [Browser-use runtime](../../../../packages/experimental/browser-use-runtime/README.md) — owns the activation rules where "who may attach" is decided today.
- [Session persistence](../../implemented/architecture/2026-06-14-session-persistence.md) — the ownership and release rules attachment reserving leans on.
