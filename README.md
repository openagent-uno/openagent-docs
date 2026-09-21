# OpenAgent documentation deployment mirror

The canonical documentation source now lives in
[`openagent/docs/site`](https://github.com/openagent-uno/openagent/tree/main/docs/site).
The public site now deploys from the canonical repository. This repository
retains the previous Pages workflow and source snapshot as transition history;
its workflow is manual-only and does not own <https://openagent.uno>.

Do not start new documentation work here. Changes belong in the `openagent`
monorepo and are mirrored here during the updater and hosting transition. This
repository will be archived, not deleted, after the public site and update chain
have been accepted; historical commits and deployment records will remain
available.
