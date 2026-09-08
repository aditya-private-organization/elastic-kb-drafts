# Kibana Options List control shows no options and applies no filters

**Summary:** An Options List control on a Kibana dashboard can silently lose its data view reference, causing the suggestion dropdown to show no values and selections to have no filtering effect on dashboard panels. Deleting and re-adding the control restores full functionality.

## Problem

A dashboard Options List control that previously worked correctly stops functioning:

- The suggestion dropdown is **empty** — no values appear when the control is clicked or typed into
- Pasting a known field value into the control has **no effect** — dashboard panels are not filtered
- The control's **edit (pencil) icon** may be unresponsive or throw an error when clicked
- Other dashboard panels continue to display data normally
- A manual KQL filter on the same field (`Add filter` in the top bar) **works correctly** — confirming the underlying data and field mapping are intact

The issue appears suddenly without any known changes to the index or data view.

## Environment

- **Product:** Kibana
- **Component:** Dashboard Controls — Options List
- **Versions:** 8.x, 9.x (Elastic Cloud Hosted / ESS)
- **Deployment type:** Elastic Cloud Hosted (ESS)

## Cause

The Options List control stores a hard-coded reference to its data view by internal ID (`dataViewId`) inside the dashboard saved object. If the data view is deleted and recreated (even with the same name), a new internal ID is assigned. The control's saved object still holds the old, now-invalid ID.

With an invalid `dataViewId`, the control cannot retrieve field values to populate the suggestion list, and the filter it generates is not applied to the dashboard. The control fails silently — no error is shown to the user.

Likely triggers include:
- A data view was deleted and recreated (e.g. during index pattern migration or cleanup)
- The data view was modified in a way that changed its internal ID
- A Kibana upgrade or saved-object migration altered the reference

> **Note:** A separate known issue covers this same symptom when it occurs specifically after copying a dashboard to a different Kibana space (see [Controls/Option List invalid reference after copying dashboard to another space](https://support.elastic.co/knowledge/6b2b4fb0)). The present article covers the general case where no space copy occurred.

## Resolution

1. Open the affected dashboard and click **Edit** (top-right).
2. Locate the broken Options List control (the one showing no suggestions).
3. Click the **gear icon** on the control → select **Delete control**.
4. Click **Add control** in the control bar → select **Options list**.
5. Configure the control:
   - **Data view:** select the correct data view for the dashboard
   - **Field:** select the keyword field that the original control was pointing to
   - Set the title, and any other options as needed
6. Click **Save and close**.
7. Click **Save** to save the dashboard.

The control will now correctly fetch values from the index and apply filters to all dashboard panels.

> **Tip:** If multiple controls on the same dashboard are affected, delete and re-add each one. Check that each control is pointing to the same data view that the dashboard panels use.

## References

- [Add controls to dashboards — Elastic Docs](https://www.elastic.co/docs/explore-analyze/visualize/add-controls)
- [Dashboard controls overview — Elastic Docs](https://www.elastic.co/docs/explore-analyze/visualize/dashboard-controls)
- [Controls/Option List invalid reference after copying dashboard to another space](https://support.elastic.co/knowledge/6b2b4fb0) — related known issue for the cross-space copy scenario

{elastic-private-context}
- Case: 02143993 — customer on Kibana 8.19 (ESS, ap-southeast-2). Dashboard "AIA File Processing". Field: `custom.data.filename` (keyword). Customer confirmed resolution: "Wow! This step seems to work" after re-adding the control. No cross-space copy was involved; no known changes to the index were reported. Root cause not lab-confirmed — stale dataViewId is the most likely explanation based on the symptom pattern and the fix behaviour.
- Related KB (cross-space variant): [6b2b4fb0](https://support.elastic.dev/knowledge/view/6b2b4fb0)
{/elastic-private-context}
