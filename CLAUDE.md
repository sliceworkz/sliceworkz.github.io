# sliceworkz.github.io

Documentation website for the Sliceworkz libraries (Jekyll, Chirpy theme): the eventstore and
eventmodeling guides live in `_posts/`, the landing pages in `_tabs/`.

## Writing the documentation

- **Describe only the current version.** The site documents the library as it is in the release it
  targets. Never talk about the past: no "Coming from 0.x?" or migration notes, no "up to 0.12",
  "since 0.9.0", "before 0.9.0", "was called", "used to", "is gone", "no longer", "is now", "renamed from",
  and no comparison with how an earlier release behaved. Explain a rule by why it holds, not by what went
  wrong before it existed. Release notes and migration guidance belong in the library repositories'
  GitHub releases, not here.
- The one place a version number belongs is where a reader needs it: the Maven version in the
  quickstarts and the version each quickstart says it uses.
- Wording about the *reader's own* history is fine — renaming their event classes, upcasting their
  legacy events, a lease's previous leader, upgrading PostgreSQL.
- Verify every signature in a code sample against the library source of the release being documented
  (github.com/sliceworkz/eventstore, github.com/sliceworkz/eventmodeling); never guess an API.
